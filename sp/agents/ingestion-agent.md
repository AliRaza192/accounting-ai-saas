# AI Ingestion Agent Specification

## 1. Agent Purpose

The AI Ingestion Agent is responsible for:
- Reading and parsing uploaded files (PDF, CSV, Excel, images)
- Performing OCR on scanned documents
- Extracting structured data
- Passing extracted data to the Classification Agent

## 2. Supported File Types

| Format | Use Cases |
|--------|-----------|
| PDF | Invoices, bills, bank statements |
| Images (JPG, PNG) | Receipts, scanned documents |
| CSV/Excel | Bank transactions, bulk imports |
| Email attachments | Various document types |

## 3. OCR Engine Configuration

### Primary: Google Cloud Vision API
- Text detection and document text recognition
- Support for multiple languages
- Handwriting recognition capability

### Fallback: Tesseract
- Local processing when cloud API unavailable
- Configured for accounting document formats
- Language-specific training models

### Output Format
All OCR results are converted to structured JSON format for downstream processing.

## 4. Data Extraction Schema

### Invoice OCR Output Structure
```json
{
  "document_type": "invoice",
  "confidence": 0.92,
  "vendor": {
    "name": "ABC Corp",
    "confidence": 0.95
  },
  "invoice_number": "INV-12345",
  "date": "2024-01-15",
  "total": 1250.00,
  "currency": "USD",
  "line_items": [
    {
      "description": "Consulting Services",
      "amount": 1000.00
    },
    {
      "description": "Tax",
      "amount": 250.00
    }
  ]
}
```

### Standardized Field Extraction
- Document type identification
- Date extraction (with format normalization)
- Currency and amount detection
- Vendor/customer information
- Line item breakdown
- Tax calculations

## 5. Validation Rules

### Confidence Thresholds
- **Reject**: Confidence < 70% (Low quality extraction)
- **Manual Review**: 70% ≤ Confidence < 85% (Human verification needed)
- **Auto-process**: Confidence ≥ 85% (Automatic processing)

### Data Validation Checks
- Required field presence verification
- Date format consistency
- Numeric value validation
- Currency format standardization
- Amount reconciliation

## 6. Error Handling

### Error Categories & Responses

#### File Format Errors
- **Issue**: Unsupported file format
- **Response**: Return error with supported format list
- **Status**: `invalid_format`

#### OCR Failures
- **Issue**: OCR processing failed
- **Response**: Log error, return partial data if available
- **Status**: `ocr_failed`

#### Corrupted Files
- **Issue**: File cannot be read/opened
- **Response**: Reject with corruption error
- **Status**: `corrupted_file`

#### Size Limit Exceeded
- **Issue**: File > 10MB
- **Response**: Reject with size limit error
- **Status**: `size_limit_exceeded`

## 7. Agent Boundaries

### CAN Do
- Read files from specified locations
- Perform OCR operations on documents
- Extract and structure data
- Queue processed data for classification

### CANNOT Do
- Modify original files in any way
- Delete or alter source documents
- Classify transactions (reserved for Classification Agent)
- Post journal entries or make accounting decisions

## 8. API Integration

### Endpoint: POST /api/v1/ai/ingest

#### Request Body
```json
{
  "file_url": "s3://bucket/invoice.pdf",
  "document_type_hint": "invoice"
}
```

#### Response Schema
```json
{
  "ingestion_id": "ing_abc123",
  "status": "completed",
  "confidence": 0.92,
  "extracted_data": {...},
  "requires_review": false
}
```

#### Status Values
- `processing`: Currently being processed
- `completed`: Processing finished successfully
- `failed`: Processing failed due to error
- `requires_manual_review`: Needs human verification

## 9. Claude API Prompts for OCR

### Document Type Identification Prompt
```
Analyze the provided document and identify the document type from these categories:
- invoice
- receipt
- bank_statement
- bill
- purchase_order
- other

Return only the document type in lowercase.
```

### Financial Data Extraction Prompt
```
Extract financial data from the following document text. Return the data in structured JSON format with these fields:
- vendor_name
- invoice_number
- date
- total_amount
- currency
- line_items (array with description and amount)

Only include fields that are clearly present in the document. If a field is not available, omit it from the response.
```

### Confidence Scoring Prompt
```
Based on the provided document image and extracted text, rate the confidence level of the data extraction on a scale of 0 to 1. Consider:
- Text clarity and readability
- Completeness of financial information
- Consistency of formatting
- Presence of key identifiers

Return only the confidence score as a decimal number.
```

## 10. Confidence Scoring Algorithm

### Base Score Calculation
```
Base Score = (
  text_clarity_score * 0.3 +
  data_completeness_score * 0.4 +
  format_consistency_score * 0.2 +
  identifier_reliability_score * 0.1
)
```

### Adjustment Factors
- **High-quality scan**: +0.1
- **Standard format**: +0.1
- **Missing key fields**: -0.2 per field
- **Poor image quality**: -0.3
- **Multiple languages**: -0.1

### Final Score Application
- Apply business rules based on document type
- Adjust for known vendor templates
- Factor in historical accuracy rates

## 11. Error Handling Flows

### Main Processing Flow
```
Start → Validate File → Choose OCR Engine → Extract Data → Validate Output → Confidence Check → Return Result
```

### Error Recovery Path
```
Error Occurred → Log Error → Determine Error Type → Apply Recovery Strategy → Return Appropriate Response
```

### Retry Logic
- OCR failures: Retry with fallback engine
- Network errors: Retry up to 3 times with exponential backoff
- Format errors: Immediate failure with descriptive message

## 12. Test Cases

### Test Case 1: Valid Invoice Processing
- **Input**: Clear PDF invoice with all required fields
- **Expected**: High confidence extraction (>0.90)
- **Output**: Complete structured data with auto-process flag

### Test Case 2: Low-Quality Image
- **Input**: Blurry JPG receipt with poor lighting
- **Expected**: Medium confidence extraction (~0.75)
- **Output**: Data with manual review flag

### Test Case 3: Unsupported Format
- **Input**: Word document (.docx)
- **Expected**: Format error response
- **Output**: Rejection with supported format list

### Test Case 4: Corrupted File
- **Input**: Partially downloaded PDF
- **Expected**: Corruption detection
- **Output**: File corruption error

### Test Case 5: Missing Key Fields
- **Input**: Invoice with missing total amount
- **Expected**: Low confidence due to incomplete data
- **Output**: Partial data with low confidence score

### Test Case 6: Multi-page Document
- **Input**: Multi-page PDF with invoice and terms
- **Expected**: Focus on relevant invoice page
- **Output**: Data from appropriate section with high confidence

### Test Case 7: Foreign Currency
- **Input**: Invoice in EUR currency
- **Expected**: Proper currency identification
- **Output**: Correct currency code and amount extraction

### Test Case 8: Large File
- **Input**: PDF > 10MB
- **Expected**: Size limit enforcement
- **Output**: Size limit exceeded error

## 13. Performance Requirements

- **Processing Time**: < 30 seconds for files under 5MB
- **Throughput**: Process 10 concurrent files
- **Accuracy**: >90% for confidence scores >0.85
- **Availability**: 99.5% uptime for API endpoints