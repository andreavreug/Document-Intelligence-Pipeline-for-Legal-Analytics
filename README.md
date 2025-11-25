# Document Intelligence Pipeline for Legal Analytics

A comprehensive solution for extracting structured information from legal and financial documents (contracts, emails, and invoices) using OCR and large language models. This pipeline automates document classification and data extraction for legal analytics workflows.

## 📋 Overview

This project processes PDF documents through a two-stage pipeline:

1. **OCR Extraction**: Converts PDF pages to images and extracts raw text using Tesseract OCR
2. **Intelligent Parsing**: Uses Databricks Foundation Models (Llama 3.3 70B) to classify documents and extract structured data in JSON format

The system is designed to handle:
- **Contracts**: Service agreements, NDAs, master agreements with terms and conditions
- **Emails**: Messages with sender/recipient information and conversational content
- **Invoices**: Billing documents with line items, totals, and payment information
- **Other**: Unclassified or unreadable documents

## 🚀 Features

- **Multi-document Processing**: Handles PDFs with multiple pages, processing each page independently
- **Intelligent Document Classification**: Automatically identifies document types based on structural patterns
- **Comprehensive Data Extraction**: 
  - Client and merchant information with email addresses
  - Invoice details (dates, amounts, line items)
  - Payment records with methods and amounts
  - Smart handling of unreadable or garbled content
- **Error Handling**: Gracefully manages OCR failures and parsing errors
- **Pandas Integration**: Outputs results as DataFrames for easy analysis and export

## 📦 Prerequisites

### Python Packages

```bash
pip install PyMuPDF pytesseract pdfplumber pillow azure-storage-blob
```

- **PyMuPDF (fitz)**: PDF manipulation and page-to-image conversion
- **pytesseract**: Python wrapper for Tesseract OCR
- **pdfplumber**: PDF parsing utility
- **Pillow**: Image processing
- **azure-storage-blob**: Azure cloud storage integration

### System Requirements

- **Tesseract OCR**: Must be installed separately on your system
  - Windows: Download from [GitHub Tesseract releases](https://github.com/UB-Mannheim/tesseract/wiki)
  - Linux: `sudo apt-get install tesseract-ocr`
  - macOS: `brew install tesseract`

### LLM Integration

- **Databricks Account**: Required for accessing Databricks Foundation Models API
- **Model**: Llama 3.3 70B Instruct endpoint configured in Databricks workspace

## 📁 Project Structure

```
Document-Intelligence-Pipeline-for-Legal-Analytics/
├── BUSA-Ingestion-LLM.ipynb    # Main Jupyter notebook with full pipeline
├── README.md                      # This file
└── case_dataset.pdf              # Sample PDF for processing (local storage)
```

## 🔧 How It Works

### Stage 1: OCR Text Extraction

The `extract_text_with_ocr()` function:
- Opens PDF using PyMuPDF
- Converts each page to a high-resolution image (default 300 DPI)
- Applies Tesseract OCR to extract text
- Combines results from all pages with page markers

**Input**: PDF file path  
**Output**: Raw extracted text with page boundaries

### Stage 2: LLM-Based Parsing

The pipeline uses an advanced prompt template that:
1. **Classifies** the document type based on structural patterns
2. **Extracts** structured fields:
   - Document metadata (type, dates)
   - Party information (client, merchant, emails, addresses)
   - Financial details (invoice amounts, subtotals, fees)
   - Line items (description, quantity, unit price, totals)
   - Payment records (method, date, amount)
3. **Validates** extracted data against expected schemas
4. **Handles** edge cases like unreadable documents

**Model**: Databricks Meta Llama 3.3 70B Instruct  
**Output**: JSON with extracted structured data

### Stage 3: Result Aggregation

Results are combined across all pages into three DataFrames:
- **Document Summary**: One row per page with document-level metadata
- **Line Items**: All extracted line items with page and merchant references
- **Payments**: All payment records with transaction details

## 💻 Usage

### Basic Usage

```python
# Configure PDF path (local or Azure Blob Storage)
pdf_path = "./case_dataset.pdf"

# Extract text using OCR
text = extract_text_with_ocr(pdf_path, dpi=300)

# Process extracted text through LLM (uses Databricks client)
# Results are automatically displayed as DataFrames
```

### Using Azure Blob Storage

Uncomment the Azure configuration in the notebook:

```python
spark.conf.set(
    "fs.azure.account.key.busa1.blob.core.windows.net",
    "YOUR_AZURE_STORAGE_KEY"
)
pdf_path = "wasbs://raw@busa1.blob.core.windows.net/case_dataset.pdf"
```

### Adjusting OCR Quality

```python
# Higher DPI = better quality but slower processing
text = extract_text_with_ocr(pdf_path, dpi=600)  # Maximum quality
text = extract_text_with_ocr(pdf_path, dpi=150)  # Faster processing
```

## 📊 Output Structure

The pipeline produces three main outputs:

### 1. Document Summary DataFrame
| Field | Type | Description |
|-------|------|-------------|
| Page | int | Page number |
| Document Type | str | "contract", "email", "invoice", or "other" |
| Client | str | Client/customer name |
| Client Email | str | Client email address |
| Merchant | str | Vendor/service provider name |
| Invoice Date | str | Issue date (invoices only) |
| Invoice Amount | float | Total amount due (invoices only) |
| Subtotal | float | Sum before fees (invoices only) |
| Other Fees | float | Additional charges (invoices only) |

### 2. Line Items DataFrame
| Field | Type | Description |
|-------|------|-------------|
| page | int | Source page |
| merchant | str | Associated merchant |
| description | str | Item description |
| quantity | float | Quantity ordered |
| unit_price | float | Price per unit |
| line_amount | float | Total for this line |

### 3. Payments DataFrame
| Field | Type | Description |
|-------|------|-------------|
| page | int | Source page |
| merchant | str | Associated merchant |
| payment_method | str | Method used (credit card, bank transfer, etc.) |
| payment_date | str | Transaction date |
| payment_amount | float | Amount paid |

## ⚠️ Important Notes

### LLM Prompt Strategy
The prompt template uses zero-temperature (temperature: 0) for consistent, deterministic results. This is critical for:
- Preventing hallucinated data
- Ensuring reproducible extractions
- Maintaining compliance with actual document content

### Field Handling Rules
- **Null values**: Used when fields are not present in document
- **Empty arrays**: Used for lists when no items found
- **Line items**: Extracted from all sources (invoices, contracts with pricing)
- **Payments**: Each distinct payment entry is extracted separately with unique amounts

### Limitations
- OCR quality depends on PDF image quality and resolution
- Complex layouts may be misinterpreted
- Handwritten text may not be extracted accurately
- Requires Tesseract OCR system installation
- Databricks API calls incur costs based on token usage

## 🔐 Security Considerations

- **API Keys**: Store Databricks credentials in environment variables, not in notebook
- **Azure Credentials**: Use managed identities or secure credential stores
- **Data**: Be aware that document content is sent to external LLM API
- **Compliance**: Ensure GDPR/HIPAA compliance if handling sensitive documents

## 📈 Performance Tips

1. **Batch Processing**: Process multiple PDFs sequentially to manage API rate limits
2. **DPI Adjustment**: Use 200-300 DPI for best balance between quality and speed
3. **Page Limits**: Consider processing large PDFs in sections
4. **Token Monitoring**: Monitor Databricks API token usage for cost optimization

## 🐛 Troubleshooting

| Issue | Solution |
|-------|----------|
| Tesseract not found | Install Tesseract OCR system package |
| Azure connection fails | Check connection string and blob storage credentials |
| JSON parse errors | Check LLM response format, may indicate API issues |
| OCR quality poor | Increase DPI or check PDF image resolution |
| Databricks API errors | Verify model endpoint exists and credentials are valid |

## 📝 Example Workflow

1. Place PDF file in local directory or Azure Blob Storage
2. Run OCR extraction to convert PDF to text
3. Process extracted text through Databricks LLM
4. Export results as CSV from DataFrames:
   ```python
   df_all_docs.to_csv("documents.csv", index=False)
   df_all_line_items.to_csv("line_items.csv", index=False)
   df_all_payments.to_csv("payments.csv", index=False)
   ```

## 🤝 Contributing

This is a BUSA Compass project at McGill University. For questions or improvements, please contact the project maintainers.

## 📄 License

[Add appropriate license here]

## 🔗 Related Resources

- [Databricks Foundation Models Documentation](https://docs.databricks.com/en/foundation-models/)
- [PyMuPDF Documentation](https://pymupdf.readthedocs.io/)
- [Tesseract OCR Documentation](https://github.com/UB-Mannheim/tesseract/wiki)
- [Azure Blob Storage Python SDK](https://learn.microsoft.com/en-us/python/api/azure-storage-blob/)

---

**Created by**: McGill BUSA Compass  
**Last Updated**: November 2025