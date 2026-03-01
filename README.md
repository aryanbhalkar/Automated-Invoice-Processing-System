# Automated-Invoice-Processing-System
<img width="1720" height="393" alt="image" src="https://github.com/user-attachments/assets/288f2bf0-26d4-47d9-806e-2c29d92eb8c7" />

# Automated Invoice Processing System (n8n Workflow)

## Overview

This n8n workflow automatically processes newly uploaded invoices from Google Drive, extracts structured invoice data using Google Gemini, stores the extracted data in Google Sheets, and generates & sends a professional email response via Gmail.

### High-Level Flow

Detect file → Download → Convert to text → Extract invoice fields → Save to Google Sheets → Generate email → Send email



## Features

* Watches a Google Drive folder for new invoice files
* Extracts structured invoice data:

  * Invoice Number
  * Client Name
  * Client Email
  * Client Address
  * Client Phone
  * Total Amount
  * Invoice Date
* Stores extracted data in a Google Sheets database
* Automatically generates a structured email response
* Sends email using Gmail
* Includes rate-limit handling with delay nodes



## Workflow Architecture (Node-by-Node)

1. **Google Drive Trigger** – Fires when a new file is created
2. **Download File** – Downloads invoice file
3. **Extract from File** – Converts invoice to text/base64
4. **Wait** – Prevents rate-limit bursts
5. **HTTP Request (Gemini API)** – First-pass extraction
6. **Edit Fields** – Cleans & structures JSON
7. **Wait1** – Rate limit protection
8. **Information Extractor (Gemini Chat Model)** – Extracts structured invoice fields
9. **Append Row in Sheet (Google Sheets)** – Stores invoice data
10. **Wait2** – Protects final LLM call
11. **AI Agent (Gemini + Structured Output Parser)** – Generates email subject + body
12. **Send a Message (Gmail)** – Sends email
13. **No Operation** – Workflow end placeholder


## Prerequisites

Before running this workflow, ensure:

### Accounts Required

* Google Drive
* Google Sheets
* Gmail
* Google Gemini API (with billing enabled)

### n8n Requirements

* n8n v1.x or later
* Proper OAuth credentials configured for:

  * Google Drive
  * Google Sheets
  * Gmail
* Gemini API key with sufficient quota


## Setup Instructions

### 1. Import the Workflow

* Download the provided `workflow.json`
* Open n8n
* Go to Workflows → Import
* Upload the JSON file

### 2. Configure Google Drive Trigger

* Set correct folder ID
* Optionally filter by file type (PDF recommended)

### 3. Configure Google Sheets

* Share spreadsheet with the credential account
* Ensure headers exist:

  * Invoice Number
  * Client Name
  * Client Email
  * Client Address
  * Client Phone
  * Total Amount
  * Invoice Date
* Use “Map each column manually”

Example mapping:

```
{{ $json.output['Invoice Number'] }}
{{ $json.output['Client Name'] }}
```

### 4. Configure Gemini API

For HTTP Request Node:

* Method: POST
* Include API key as query parameter
* Ensure prompt forces **pure JSON output**
* Add retry with exponential backoff

For Gemini Chat Nodes:

* Ensure invoice text is passed (not placeholder text)
* Enable structured output parsing

### 5. Configure Gmail Node

Subject:

```
{{ $json.response.output.Subject }}
```

Body:

```
{{ $json.response.output.Email }}
```

Use the expression picker to verify correct output path.


## Recommended Invoice Format

For best results:

* Use searchable PDFs (text layer)
* Use high-resolution scanned images
* Avoid low-quality scanned PDFs without OCR

If scanned PDFs are image-only, OCR should be added before extraction.


## Rate Limit & Reliability Strategy

This workflow makes multiple Gemini API calls. To prevent 429 errors:

* Use Wait nodes between LLM calls
* Enable retries with exponential backoff:

  * Attempt 1 → 2s
  * Attempt 2 → 5s
  * Attempt 3 → 10s
  * Attempt 4 → 20s
* Limit workflow concurrency (queue mode recommended)
* Avoid unnecessary duplicate LLM calls

If error says:

> "Please retry in XX seconds"

Use that value as delay before retrying.


## Troubleshooting

### 429 Too Many Requests

* Reduce concurrency
* Increase wait times
* Check Gemini quota/billing

### JSON Parsing Errors

* Remove ```json code fences from model output
* Ensure Gemini returns pure JSON
* Confirm JSON.parse receives a string

### Email Content Incorrect

* Verify AI Agent prompt receives actual extracted values
* Confirm Gmail node maps to structured parser output
* Ensure you're not using template/example JSON instead of live output

### Google Sheets Permission Error

* Share spreadsheet with n8n credential account
* Confirm edit access



## Mapping Cheat Sheet

| Component     | Field              | Source                                 |
| ------------- | ------------------ | -------------------------------------- |
| Google Sheets | Invoice Number     | `{{ $json.output['Invoice Number'] }}` |
| Google Sheets | Client Name        | `{{ $json.output['Client Name'] }}`    |
| AI Agent      | All invoice fields | Information Extractor output           |
| Gmail Subject | Subject            | `{{ $json.response.output.Subject }}`  |
| Gmail Body    | Email              | `{{ $json.response.output.Email }}`    |

If output path differs, use n8n’s expression picker.

---

## Security Notes

* Do NOT commit API keys to GitHub
* Do NOT export credentials
* Use environment variables for:

  * GEMINI_API_KEY
  * SHEET_ID
  * DRIVE_FOLDER_ID
