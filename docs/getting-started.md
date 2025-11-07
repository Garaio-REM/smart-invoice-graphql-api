# Getting Started with Smart.Invoice GraphQL API

This guide will help you get started with the Smart.Invoice GraphQL API, from requesting access to making your first API call.

## Prerequisites

- Access to Smart.Invoice
- Contact with your Smart.Invoice administrator
- Basic understanding of REST and GraphQL APIs
- A tool for making API requests (curl, Bruno, Postman, or your programming language of choice)

## Step 1: Request API Access

API access is managed by your Smart.Invoice administrator. To get started:

1. Contact your Smart.Invoice administrator
2. Request API access for your tenant
3. Provide a brief description of your use case
4. Receive your unique API key

**Important**: Keep your API key secure! Treat it like a password and never share it publicly or commit it to version control.

## Step 2: Understand the API Endpoints

The Smart.Invoice API has two environments:

### Test Environment

**Base URL**: `https://smart-invoice-test.fly.dev`

Use for development and testing your integration.

### Production Environment

**Base URL**: `https://smart-invoice.fly.dev`

Use for production access to live invoice data.

### API Endpoints

Both environments provide the same endpoints:

**Token Endpoint**: `POST /api/token`

- Exchange your API key for a JWT access token
- Authentication: API key in request body

**GraphQL Endpoint**: `POST /api/graphql`

- Query invoice data
- Authentication: JWT token in Authorization header

## Step 3: Get Your First Access Token

Once you have your API key, exchange it for an access token:

```bash
curl -X POST https://smart-invoice.fly.dev/api/token \
  -H "Content-Type: application/json" \
  -d '{
    "grant_type": "app_key",
    "app_key": "your-api-key-here"
  }'
```

**Successful Response (200 OK):**

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJhdmtfc2VydmljZSIsImV4cCI6MTczMDk4NDg5NSwiaWF0IjoxNzMwOTgxMjk1LCJpc3MiOiJhdmtfc2VydmljZSIsImp0aSI6IjIxM2MyYzJiLTU4MmMtNGI1My05ZjkzLWVjOGNjYzVmNWNhMiIsIm5iZiI6MTczMDk4MTI5NCwic3ViIjoie1widGVuYW50X2lkXCI6MSxcInR5cGVcIjpcImFwaV9jb25zdW1lclwifSIsInR5cCI6ImFjY2VzcyJ9.Xb7vR9YqV8oN2_3PdF4mL6xJ8wK9eT5iU1cA0bH2gVo"
}
```

## Step 4: Make Your First GraphQL Query

Use the access token to query invoices:

```bash
curl -X POST https://smart-invoice.fly.dev/api/graphql \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query { invoices(limit: 5) { page { id originalFilename state processedAt } } }"
  }'
```

**Successful Response (200 OK):**

```json
{
  "data": {
    "invoices": {
      "page": [
        {
          "id": 123,
          "originalFilename": "invoice-2024-001.pdf",
          "state": "OK",
          "processedAt": "2024-11-07T10:30:00Z"
        },
        {
          "id": 124,
          "originalFilename": "invoice-2024-002.pdf",
          "state": "OK",
          "processedAt": "2024-11-07T11:15:00Z"
        }
      ]
    }
  }
}
```

## Step 5: Implement Token Refresh Logic

Access tokens expire after **1 hour** (3600 seconds). Your application should:

1. Store the token securely in memory
2. Implement automatic token refresh before expiration
3. Handle 401 Unauthorized responses by requesting a new token

**Example token management pattern (Python):**

```python
import requests
from datetime import datetime, timedelta

class SmartInvoiceClient:
    def __init__(self, api_url, api_key):
        self.api_url = api_url
        self.api_key = api_key
        self.token = None
        self.token_expires_at = None

    def get_token(self):
        """Get a fresh access token"""
        response = requests.post(
            f"{self.api_url}/api/token",
            json={"grant_type": "app_key", "app_key": self.api_key}
        )
        response.raise_for_status()

        self.token = response.json()["access_token"]
        # Token expires in 1 hour, refresh 5 minutes before
        self.token_expires_at = datetime.now() + timedelta(minutes=55)

        return self.token

    def ensure_token(self):
        """Ensure we have a valid token"""
        if not self.token or datetime.now() >= self.token_expires_at:
            return self.get_token()
        return self.token

    def query_invoices(self, query):
        """Query the GraphQL API"""
        token = self.ensure_token()

        response = requests.post(
            f"{self.api_url}/api/graphql",
            headers={"Authorization": f"Bearer {token}"},
            json={"query": query}
        )
        response.raise_for_status()

        return response.json()
```

## Next Steps

Now that you've made your first API call:

1. **Explore the API** - See [GraphQL API Reference](graphql-api.md) for all available fields
2. **Test with Bruno** - Set up Bruno for easier testing: [Testing with Bruno](testing-with-bruno.md)
3. **Learn patterns** - Check out common use cases: [Examples & Use Cases](examples.md)
4. **Understand auth** - Deep dive into security: [Authentication](authentication.md)

## Common Issues

### "Invalid API key" error

- Verify you're using the correct API key
- Ensure there are no extra spaces or newlines
- Contact your administrator to verify the key is still active

### "Unauthorized" error on GraphQL endpoint

- Check that you're including the Authorization header
- Verify the token hasn't expired (tokens last 1 hour)
- Request a fresh token if needed

### Rate limit exceeded

- You're making too many failed authentication attempts
- Wait 1 hour before retrying
- Check your API key is correct to avoid further failures

## Support

If you encounter issues or have questions:

- Contact your Smart.Invoice administrator
- Refer to the [GraphQL API Reference](graphql-api.md)
- Check the [Testing with Bruno](testing-with-bruno.md) guide
