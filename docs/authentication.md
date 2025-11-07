# Authentication

The Smart.Invoice GraphQL API uses a two-step authentication process with API keys and JWT tokens.

## Overview

```
┌──────────┐                    ┌──────────────┐                ┌─────────────┐
│  Client  │                    │  API Token   │                │  GraphQL    │
│          │                    │  Endpoint    │                │  Endpoint   │
└─────┬────┘                    └──────┬───────┘                └──────┬──────┘
      │                                │                               │
      │  1. POST /api/token            │                               │
      │  { api_key }                   │                               │
      │ ─────────────────────────────> │                               │
      │                                │                               │
      │  2. { access_token }           │                               │
      │ <───────────────────────────── │                               │
      │                                │                               │
      │  3. POST /api/graphql                                          │
      │  Authorization: Bearer token                                   │
      │ ─────────────────────────────────────────────────────────────> │
      │                                │                               │
      │  4. { data }                                                   │
      │ <───────────────────────────────────────────────────────────── │
      │                                │                               │
```

## Authentication Flow

### Step 1: API Key Generation (Administrator)

Your Smart.Invoice administrator generates an API key for your tenant:

1. Admin logs into Smart.Invoice admin panel
2. Navigates to tenant configuration
3. Generates a new API key
4. Securely shares the key with you

**Important**: API keys are hashed and stored securely. The administrator cannot retrieve the key after generation - if lost, a new key must be generated.

### Step 2: Token Exchange (Your Application)

Exchange your API key for a JWT access token:

**Request:**

```http
POST /api/token
Content-Type: application/json

{
  "grant_type": "app_key",
  "app_key": "your-api-key-here"
}
```

**Success Response (200 OK):**

```json
{
  "access_token": "eyJhbGc...iOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Error Responses:**

| Status | Error | Description |
|--------|-------|-------------|
| 400 | `invalid_grant` | Invalid or expired API key |
| 400 | `unsupported_grant_type` | Only `app_key` is supported |
| 400 | `invalid_request` | Missing required parameters |
| 429 | `rate_limit_exceeded` | Too many failed attempts |

### Step 3: API Request with Token

Include the access token in the Authorization header:

**Request:**

```http
POST /api/graphql
Authorization: Bearer eyJhbGc...iOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "query": "query { invoices(limit: 10) { page { id } } }"
}
```

**Success Response (200 OK):**

```json
{
  "data": {
    "invoices": {
      "page": [...]
    }
  }
}
```

**Error Responses:**

| Status | Description |
|--------|-------------|
| 401 | Missing or invalid Authorization header |
| 401 | Expired or invalid token |
| 401 | Tenant no longer exists |

## Token Details

### Token Lifetime

- **Duration**: 1 hour (3600 seconds)
- **Refresh**: No refresh tokens - request a new token when needed
- **Recommendation**: Refresh 5 minutes before expiration

## Rate Limiting

To protect against brute force attacks, the API implements rate limiting:

### Token Endpoint Rate Limits

- **Limit**: 3 failed authentication attempts per hour
- **Scope**: Per IP address
- **Duration**: 1 hour lockout after exceeding limit
- **Note**: Successful authentications do NOT count toward the limit

**Rate Limit Response (429 Too Many Requests):**

```json
{
  "error": "rate_limit_exceeded",
  "error_description": "Too many authentication attempts. Please try again later."
}
```

**Best Practices:**

- Store your API key securely and avoid failed attempts
- Implement exponential backoff for retries
- Monitor your authentication success rate

## Security Features

### Data Isolation

- Each token is bound to a specific tenant
- The API automatically filters all queries to your tenant's data
- You cannot access data from other tenants, even with a valid token

### API Key Security

- API keys are hashed using SHA-256 before storage
- Never stored in plaintext
- Cannot be retrieved after generation
- Administrator must generate a new key if lost

### Transport Security

- All API requests **must** use HTTPS
- HTTP requests are not accepted
- Tokens are transmitted securely in headers

### Token Validation

Every request validates:

1. Token signature is valid
2. Token has not expired
3. Tenant referenced in token still exists
4. Token type is correct (access token)

## Best Practices

### 1. Store Credentials Securely

```bash
# Use environment variables
export SMART_INVOICE_API_KEY="your-api-key"
export SMART_INVOICE_API_URL="https://your-instance.smart-invoice.ch"
```

```python
import os

api_key = os.environ.get("SMART_INVOICE_API_KEY")
api_url = os.environ.get("SMART_INVOICE_API_URL")
```

### 2. Implement Token Caching

Don't request a new token for every API call:

```javascript
class TokenManager {
  constructor(apiUrl, apiKey) {
    this.apiUrl = apiUrl;
    this.apiKey = apiKey;
    this.token = null;
    this.expiresAt = null;
  }

  async getToken() {
    // Return cached token if still valid
    if (this.token && this.expiresAt > Date.now() + 5 * 60 * 1000) {
      return this.token;
    }

    // Request new token
    const response = await fetch(`${this.apiUrl}/api/token`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        grant_type: 'app_key',
        app_key: this.apiKey
      })
    });

    const data = await response.json();
    this.token = data.access_token;
    this.expiresAt = Date.now() + 55 * 60 * 1000; // 55 minutes

    return this.token;
  }
}
```

### 3. Handle Token Expiration

```python
def make_api_request(client, query):
    """Make API request with automatic token refresh on 401"""
    try:
        response = client.graphql_request(query)
        return response
    except UnauthorizedError:
        # Token expired, get a new one
        client.refresh_token()
        # Retry the request
        return client.graphql_request(query)
```

### 4. Never Commit Secrets

Add to `.gitignore`:

```gitignore
.env
.env.local
config/secrets.yml
credentials.json
```

## Troubleshooting

### "Invalid API key" Error

**Possible causes:**

- API key is incorrect or has typos
- API key was revoked by administrator
- Whitespace or newlines in the API key

**Solution:**

- Verify the API key is correct
- Check for extra spaces/newlines
- Contact administrator to verify key status

### "Token expired" Error

**Possible causes:**

- Token is older than 1 hour
- System clock is out of sync

**Solution:**

- Request a new token
- Verify system time is accurate
- Implement automatic token refresh

### Rate Limit Exceeded

**Possible causes:**

- Too many failed authentication attempts
- Multiple services using the same IP

**Solution:**

- Wait 1 hour before retrying
- Verify API key is correct
- Check for misconfigured services

## Token Validation Example

You can decode (but not verify without the secret) JWT tokens using tools like jwt.io:

```bash
# Decode token header and payload (does not verify signature)
echo "eyJhbGc..." | base64 -d
```

**Note**: Never trust decoded JWT data without signature verification. The Smart.Invoice API handles all verification automatically.

## Next Steps

- [GraphQL API Reference](graphql-api.md) - Learn about available queries
- [Testing with Bruno](testing-with-bruno.md) - Set up authentication in Bruno
- [Getting Started](getting-started.md) - Initial setup guide
