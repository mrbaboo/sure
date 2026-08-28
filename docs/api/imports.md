# Imports API Documentation

The Imports API allows external applications to programmatically upload and process financial data from CSV files and PDF bank or credit-card statements. This API supports creating transaction imports, configuring column mappings, triggering the import process, and submitting PDF statements for AI extraction.

## Authentication requirements

All import endpoints require an OAuth2 access token or API key that grants the appropriate scope (`read` or `read_write`).

## Available endpoints

| Endpoint | Scope | Description |
| --- | --- | --- |
| `GET /api/v1/imports` | `read` | List imports with filtering and pagination. |
| `GET /api/v1/imports/{id}` | `read` | Retrieve a single import with configuration and statistics. |
| `POST /api/v1/imports` | `read_write` | Create a new import and optionally trigger processing. |

## Filtering options

The `GET /api/v1/imports` endpoint supports the following query parameters:

| Parameter | Type | Description |
| --- | --- | --- |
| `page` | integer | Page number (default: 1) |
| `per_page` | integer | Items per page (default: 25, max: 100) |
| `status` | string | Filter by status: `pending`, `importing`, `complete`, `failed`, `reverting`, `revert_failed` |
| `type` | string | Filter by import type: `TransactionImport`, `TradeImport`, etc. |

## Import object

An import response includes configuration and processing statistics:

```json
{
  "data": {
    "id": "uuid",
    "type": "TransactionImport",
    "status": "pending",
    "created_at": "2024-01-15T10:30:00Z",
    "updated_at": "2024-01-15T10:30:00Z",
    "account_id": "uuid",
    "configuration": {
      "date_col_label": "date",
      "amount_col_label": "amount",
      "name_col_label": "name",
      "category_col_label": "category",
      "tags_col_label": "tags",
      "notes_col_label": "notes",
      "account_col_label": null,
      "date_format": "%m/%d/%Y",
      "number_format": "1,234.56",
      "signage_convention": "inflows_positive"
    },
    "stats": {
      "rows_count": 150,
      "valid_rows_count": 150
    }
  }
}
```

## Creating an import

When creating an import, you must provide the file content and the column mappings.

### Parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `raw_file_content` | string | The raw CSV content as a string. |
| `file` | file | Alternatively, the CSV file can be uploaded as a multipart form-data part. A PDF here creates a PDF statement import instead - see below. |
| `account_id` | uuid | Optional. The ID of the account to import into. |
| `date_col_label` | string | The header name for the date column. |
| `amount_col_label` | string | The header name for the amount column. |
| `name_col_label` | string | The header name for the transaction name column. |
| `publish` | boolean | If `true`, the import will be automatically queued for processing if configuration is valid. |

Example request body:

```json
{
  "raw_file_content": "date,amount,name\n01/01/2024,10.00,Test",
  "date_col_label": "date",
  "amount_col_label": "amount",
  "name_col_label": "name",
  "account_id": "uuid",
  "publish": "true"
}
```

## Creating a PDF statement import

Upload a bank or credit-card statement as a PDF and Sure stores it as an account
statement, creates a `PdfImport`, and queues AI extraction of its transactions - the
same flow as dropping a PDF on the Imports page. This needs an AI provider configured in
Sure (Settings > Self-hosting); without one the import ends `failed` with the provider
error in `error`.

### Parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `file` | file | **Required.** The statement, as a `multipart/form-data` part with content type `application/pdf`. Maximum 25MB. |
| `type` | string | `PdfImport`. Optional: a PDF `file` is treated as a `PdfImport` whatever `type` says. |
| `account_id` | uuid | Optional. Link the statement to this account up front; otherwise Sure suggests one from the statement and it is chosen during review. |

The API key's user must be a family admin or member (the same rule as the web uploader).

```bash
curl -X POST https://sure.example/api/v1/imports \
  -H "X-Api-Key: $SURE_API_KEY" \
  -F type=PdfImport \
  -F file=@statement.pdf
```

### Responses

| Status | Meaning |
| --- | --- |
| `201` | Created. `data.status` is `importing` while extraction runs. |
| `200` | This PDF was already uploaded by someone in your family and its import is still usable; that import is returned (same `id`) and no second statement is stored. If the earlier import had failed, the stored statement is reused but a fresh import is created (`201`). |
| `403` | `forbidden` - the key's user cannot manage statements. |
| `404` | `not_found` - `account_id` is not an account the user can access. |
| `409` | `duplicate_statement` - already stored as a statement the user cannot access; or `account_locked` - the existing import has been published and cannot be moved to `account_id`. |
| `422` | `missing_content`, `invalid_file_type` (not `application/pdf` or not a PDF), `invalid_pdf`, or `file_too_large`. |

### Following progress

Poll `GET /api/v1/imports/{id}`:

- `importing` - AI extraction in progress
- `pending` - transactions extracted and waiting for review; publish them in Sure's UI
  (or via `POST /api/v1/imports/{id}/publish` where available)
- `complete` - nothing left to review (a non-statement PDF, or every transaction was
  already recorded)
- `failed` - see `error`

## Error responses

Errors conform to the shared `ErrorResponse` schema:

```json
{
  "error": "error_code",
  "message": "Human readable error message",
  "errors": ["Optional array of validation errors"]
}
```

