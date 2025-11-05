# Async Background Check Flow

> Documentation for asynchronous data source execution flow, primarily used for Background Check but applicable to any data source that requires async operation.

## Overview

Background Check reports and certain other data sources are **asynchronous by design** because they execute multiple online data sources that can take several minutes to complete. This document describes the async execution pattern used by these APIs.

### When to Use Async Pattern

- **Background Check** (pessoa-fisica and pessoa-juridica): Always async
- **Other data sources**: When execution time is expected to exceed a few seconds
- **Client preference**: When clients want to call any data source asynchronously

## API Authentication

**Current Standard (Updated)**:
- Method: `POST`
- Token location: HTTP Header
- Header name: `token`

```bash
--header 'token: YOUR-TOKEN-HERE'
```

> **Note**: Legacy documentation may show token as query parameter. The header-based approach is now the recommended standard.

## Async Execution Flow

The async pattern consists of two distinct API calls:

### 1. Initial Request (Trigger Execution)

**Purpose**: Start the async execution and obtain a unique transaction identifier (UID)

**Endpoint Example** (Pessoa Física):
```bash
POST https://api.exato.digital/br/exato/background-check/pessoa-fisica.json?cpf=29481162877
Header: token: YOUR-TOKEN-HERE
```

**Response**:
- `TransactionResultTypeCode`: `12` (AsyncExecutionInProgress)
- `UniqueIdentifier`: Unique transaction ID (UID)

**Example Response**:
```json
{
  "TransactionResultTypeCode": 12,
  "TransactionResultType": "AsyncExecutionInProgress",
  "ApiResultType": "async_execution_in_progress",
  "UniqueIdentifier": "abc123def456ghi789jkl012",
  "Message": "Async execution in progress"
}
```

### 2. Polling Request (Check Status)

**Purpose**: Monitor execution progress and retrieve results when complete

**Endpoint**:
```bash
GET/POST https://api.exato.digital/br/exato/background-check/pessoa-fisica.json?uid=abc123def456ghi789jkl012
Header: token: YOUR-TOKEN-HERE
```

**While Processing**:
- Returns `TransactionResultTypeCode: 12` (still executing)
- Continue polling every 30 seconds

**When Complete**:
- Returns final status code (see Return Codes section)
- Payload available in `Result` field
- PDF report available via `PdfUrl` field

## Response Fields

All API responses include the following standard fields:

| Field | Type | Description |
|-------|------|-------------|
| `TransactionResultTypeCode` | number | Numeric status code (1, 2, 5, 9, 10, 12, 255) |
| `TransactionResultType` | string | Human-readable status (PascalCase) |
| `ApiResultType` | string | Machine-readable status (snake_case) |
| `UniqueIdentifier` | string | Unique transaction ID (UID) |
| `Message` | string | Descriptive message |
| `Date` | string | ISO 8601 timestamp |
| `ElapsedTimeInMilliseconds` | number | Total execution time |
| `HasPdf` | boolean | Whether PDF report is available |
| `PdfUrl` | string | URL to download PDF report (when available) |
| `OriginalFilesUrl` | string | URL to download original source files (when available) |
| `TotalCostInCredits` | number | Credits consumed by this transaction |
| `BalanceInCredits` | number | Remaining credit balance |
| `ValidationResultRiskIndicator` | string | Overall risk indicator: `"green"`, `"amber"`, or `"red"` (Background Check only) |
| `Result` | object | Complete data payload from all executed data sources |

## Return Codes (TransactionResultTypeCode)

The API returns two fields for result type identification:
- **`TransactionResultType`**: Human-readable PascalCase value (e.g., `"SuccessWithRemarks"`)
- **`ApiResultType`**: Machine-readable snake_case value (e.g., `"success_with_remarks"`)

### Success Codes

| Code | TransactionResultType | ApiResultType | Description | Action Required |
|------|-----------------------|---------------|-------------|-----------------|
| `1` | `Success` | `success` | Report executed successfully | Store payload and download PDF evidence |
| `2` | `SuccessWithRemarks` | `success_with_remarks` | Executed with some warnings (e.g., temporary data source unavailable) | Store payload and download PDF evidence |

### Execution in Progress

| Code | TransactionResultType | ApiResultType | Description | Action Required |
|------|-----------------------|---------------|-------------|-----------------|
| `12` | `AsyncExecutionInProgress` | `async_execution_in_progress` | Transaction still executing | Continue polling |

### Data Issues

| Code | TransactionResultType | ApiResultType | Description | Action Required |
|------|-----------------------|---------------|-------------|-----------------|
| `5` | `EntityNotFound` | `entity_not_found` | CPF does not exist or is a minor | Log and handle per business rules |

### Retry-able Errors

| Code | TransactionResultType | ApiResultType | Description | Action Required |
|------|-----------------------|---------------|-------------|-----------------|
| `9` | `Timeout` | `timeout` | Execution timeout reached | Retry from step 1 (initial request) |
| `10` | `AttemptsLimitReached` | `attempts_limit_reached` | Maximum retry attempts exceeded | Retry from step 1 (initial request) |

### System Errors

| Code | TransactionResultType | ApiResultType | Description | Action Required |
|------|-----------------------|---------------|-------------|-----------------|
| `255` | `InternalError` | `internal_error` | Internal system error | Contact Exato support |
| Other | - | - | Unexpected codes | Log and contact Exato if persistent |

> **Billing Note**: Transactions returning error codes (9, 10, 255, etc.) are not charged.

## Implementation Recommendations

### Polling Strategy

```
1. Make initial POST request to trigger execution
2. Extract UniqueIdentifier from response
3. Wait 30 seconds
4. Poll with UID using GET/POST request
5. If TransactionResultTypeCode == 12:
   - Wait 30 seconds
   - Go to step 4
6. Otherwise, process final result
```

### Average Execution Time

- **Typical**: ~5 minutes
- **Maximum**: Configurable timeout (can be adjusted per client)

### Data Retention

**Client Responsibility**:
- Store complete payload (`Result` field)
- Download and archive PDF evidence (`PdfUrl`)
- Reason: Exato archives data after 6 months

### PDF Report Details

**PDF Password**: First 6 digits of issuer CNPJ
- Default: `069751` (from CNPJ 06975199000150)

**Download**: Binary download from `PdfUrl` field

## Risk Indicators

### ValidationResultRiskIndicator

Background Check reports include an overall risk indicator based on all executed data sources:

| Indicator | Color | Priority |
|-----------|-------|----------|
| `red` | Red | Highest (overrides all) |
| `amber` | Yellow | Medium (overrides green) |
| `green` | Green | Lowest |

**Priority Logic**:
- If any indicator is `red` → Final result is `red`
- Else if any indicator is `amber` → Final result is `amber`
- Otherwise → Final result is `green`

### Important Notes

- **Do not parse individual data blocks**: Due to complexity and frequent changes in data sources
- **Use aggregated indicators**: Exato can customize indicators based on client-specific business rules
- **Contact Exato**: For custom indicator creation or modifications

## Complete Example Flow

### Step 1: Initial Request

```bash
curl --location --request POST 'https://api.exato.digital/br/exato/background-check/pessoa-fisica.json?cpf=99999999999' \
--header 'token: YOUR-TOKEN-HERE'
```

**Response**:
```json
{
  "TransactionResultTypeCode": 12,
  "TransactionResultType": "AsyncExecutionInProgress",
  "ApiResultType": "async_execution_in_progress",
  "UniqueIdentifier": "abc123def456ghi789jkl012",
  "Message": "Background check execution started"
}
```

### Step 2: Polling (while executing)

```bash
curl --location 'https://api.exato.digital/br/exato/background-check/pessoa-fisica.json?uid=abc123def456ghi789jkl012' \
--header 'token: YOUR-TOKEN-HERE'
```

**Response** (still processing):
```json
{
  "TransactionResultTypeCode": 12,
  "TransactionResultType": "AsyncExecutionInProgress",
  "ApiResultType": "async_execution_in_progress",
  "UniqueIdentifier": "abc123def456ghi789jkl012",
  "Message": "Execution in progress, please retry in 30 seconds"
}
```

### Step 3: Final Result

**Response** (completed):
```json
{
  "TransactionResultTypeCode": 1,
  "TransactionResultType": "Success",
  "ApiResultType": "success",
  "UniqueIdentifier": "abc123def456ghi789jkl012",
  "ValidationResultRiskIndicator": "green",
  "PdfUrl": "https://api.exato.digital/pdf/download/abc123def456ghi789jkl012.pdf",
  "HasPdf": true,
  "TotalCostInCredits": 1,
  "BalanceInCredits": 999987,
  "ElapsedTimeInMilliseconds": 287000,
  "Date": "2025-11-05T16:24:14-03:00",
  "Result": {
    "CPF": "29481162877",
    "Nome": "João da Silva"
  }
}
```

> **Note**: The `Result` field contains the complete data payload from all executed data sources. The structure shown above is simplified for brevity.

## API Catalog and Additional Resources

### Swagger Documentation
- Full API catalog: https://api.exato.digital/swagger-ui/

### Help Center
- Integration guide: https://help.exato.digital/pt/articles/5860451-informacoes-importantes-antes-da-sua-integracao

> **Note**: Some help center articles may contain outdated authentication methods (query parameter token). Always use header-based token as shown in this document.

## Testing

### Browser Testing
Both initial and polling requests can be tested directly in browser for quick validation during development.

### Recommended Test Flow
1. Make initial POST request
2. Copy returned UID
3. Poll with UID every 30 seconds
4. Verify final payload and PDF download

## Client Integration Best Practices

When integrating with the async API, clients should:

- **Polling interval**: Use 30-second fixed intervals (exponential backoff optional but not required)
- **Error handling**: Handle all return codes gracefully per the Return Codes section
- **Audit trail**: Store complete API responses for compliance and debugging
- **PDF download**: Download and archive PDF evidence immediately upon success
- **Retry logic**: Implement automatic retry for codes 9 (Timeout) and 10 (AttemptsLimitReached)
- **Timeout configuration**: Implement client-side timeout (recommended: 10 minutes minimum)
- **Logging**: Log all UID values for request traceability

## Appendix: Pessoa Jurídica Example

Background Check for legal entities follows the same pattern:

```bash
# Initial request
POST https://api.exato.digital/br/exato/background-check/pessoa-juridica.json?cnpj=12387530000113
Header: token: YOUR-TOKEN-HERE

# Polling
GET https://api.exato.digital/br/exato/background-check/pessoa-juridica.json?uid={UID}
Header: token: YOUR-TOKEN-HERE
```

All return codes, risk indicators, and implementation recommendations apply equally to both PF and PJ variants.
