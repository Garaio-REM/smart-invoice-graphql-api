# Smart.Invoice GraphQL API Documentation

Welcome to the Smart.Invoice GraphQL API documentation. This API allows you to programmatically access and retrieve invoice data from the Smart.Invoice platform.

## Overview

The Smart.Invoice GraphQL API provides:

- **Flexible Queries**: GraphQL interface for precise data retrieval
- **Incremental Sync**: Fetch only new invoices since your last sync
- **Pagination**: Handle large datasets efficiently
- **Tenant Isolation**: Automatic data filtering for your organization

## API Endpoints

### Test Environment

**Base URL**: `https://smart-invoice-test.fly.dev`

Use this environment for development and testing.

### Production Environment

**Base URL**: `https://smart-invoice.fly.dev`

Use this environment for production integrations.

## Quick Start

1. **Request API Access** - Contact your Smart.Invoice administrator to obtain an API key
2. **Get an Access Token** - Exchange your API key for a JWT token
3. **Query the API** - Use the token to retrieve invoice data via GraphQL

```bash
# 1. Get access token
curl -X POST https://smart-invoice.fly.dev/api/token \
  -H "Content-Type: application/json" \
  -d '{"grant_type": "app_key", "app_key": "your-api-key"}'

# 2. Query invoices (use the token from step 1)
curl -X POST https://smart-invoice.fly.dev/api/graphql \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query": "query { invoices(limit: 10) { page { id originalFilename state } } }"}'
```

## Documentation

- **[Getting Started](docs/getting-started.md)** - How to obtain API access and set up authentication
- **[Authentication](docs/authentication.md)** - Detailed authentication flow and security
- **[GraphQL API Reference](docs/graphql-api.md)** - API documentation with pagination and incremental sync
- **[Testing with Bruno](docs/testing-with-bruno.md)** - How to test the API using Bruno REST client

## Key Features

### Incremental Synchronization

Efficiently sync only new invoices since your last fetch:

```graphql
query {
  invoices(limit: 50, sinceId: 1234) {
    page {
      id
      originalFilename
      state
      processedAt
    }
  }
}
```

### Pagination

Handle large datasets with cursor-based pagination:

```graphql
query {
  invoices(limit: 20, cursor: "encoded_cursor") {
    page { id originalFilename }
    cursor
  }
}
```

### Invoice States

The API returns invoices in final processing states:

- `OK` - Successfully processed
- `REJECTED` - Processing failed (see `errorReason` for details)
- `ERROR` - System error during processing

## Support

For API access requests or technical support, please contact your Smart.Invoice administrator.

## Rate Limiting

The API implements rate limiting to ensure fair usage:

- **Token endpoint**: 3 failed authentication attempts per hour per IP address
- **GraphQL endpoint**: Contact your administrator for current limits

Exceeding rate limits will result in `429 Too Many Requests` responses.
