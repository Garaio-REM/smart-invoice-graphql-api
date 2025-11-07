# GraphQL API Reference

Documentation for the Smart.Invoice GraphQL API.

## Endpoints

### Test Environment

```
POST https://smart-invoice-test.fly.dev/api/graphql
```

Use for development and testing.

### Production Environment

```
POST https://smart-invoice.fly.dev/api/graphql
```

Use for production integrations.

## Authentication

All requests require a valid JWT token in the Authorization header:

```http
Authorization: Bearer YOUR_ACCESS_TOKEN
```

See [Authentication](authentication.md) for details on obtaining a token.

## Schema Discovery with Introspection

GraphQL is self-documenting through introspection queries. You can discover all available types, fields, and their descriptions directly from the API.

### Using GraphQL Tools

Most GraphQL clients and tools support introspection automatically:

- **Bruno**: GraphQL mode provides autocomplete and field suggestions
- **GraphiQL/GraphQL Playground**: Built-in schema explorer
- **Insomnia**: Automatic schema documentation
- **Apollo Client/Relay**: Schema-aware at build time

### Manual Introspection Query

To explore the schema programmatically:

```graphql
query IntrospectSchema {
  __schema {
    types {
      name
      description
      fields {
        name
        description
        type {
          name
          kind
        }
      }
    }
  }
}
```

### Query Specific Type

To see details about a specific type (e.g., Invoice):

```graphql
query IntrospectInvoice {
  __type(name: "Invoice") {
    name
    description
    fields {
      name
      description
      type {
        name
        kind
        ofType {
          name
          kind
        }
      }
    }
  }
}
```

## Main Query: invoices

The API provides a single main query to retrieve invoices:

```graphql
invoices(
  limit: Int
  cursor: String
  sinceId: Int
): InvoiceConnection!
```

**Parameters:**

- `limit` - Number of invoices to return (default: 20, max: 100)
- `cursor` - Pagination cursor from previous response (for next page)
- `sinceId` - Return only invoices with ID greater than this value (for incremental sync)

**Returns:** `InvoiceConnection` with `page` (array of invoices) and `cursor` (for pagination)

**Note:** Only returns invoices in final states: `OK`, `REJECTED`, or `ERROR`.

## Pagination

The API uses cursor-based pagination for efficient data retrieval.

### How It Works

1. Make initial request without cursor
2. Response includes `cursor` field
3. Use cursor in next request to get next page
4. Repeat until cursor is `null` (no more pages)

### Example: Paginated Retrieval

```graphql
# First request
query {
  invoices(limit: 20) {
    page {
      id
      originalFilename
      state
    }
    cursor
  }
}

# Response includes cursor: "eyJpZCI6MjB9"

# Second request with cursor
query {
  invoices(limit: 20, cursor: "eyJpZCI6MjB9") {
    page {
      id
      originalFilename
      state
    }
    cursor
  }
}
```

### Best Practices

- Use reasonable page sizes (20-100 items)
- Store cursor for resuming later
- Handle `null` cursor (end of results)
- Don't decode or manipulate cursor strings - they are opaque tokens

## Incremental Synchronization

Use `sinceId` to fetch only new invoices since your last sync.

### How It Works

1. **Initial sync**: Fetch all invoices, track highest ID
2. **Subsequent syncs**: Pass highest ID as `sinceId`
3. **Receive**: Only invoices with ID > `sinceId`

This is more efficient than cursor-based pagination when you only need new records.

### Example: Incremental Sync

```graphql
# Initial sync - get all invoices
query {
  invoices(limit: 100) {
    page {
      id
      originalFilename
      state
      processedAt
    }
  }
}

# Response: invoices with IDs 1-100
# Store: lastSyncId = 100

# Later sync - get only new invoices
query {
  invoices(limit: 100, sinceId: 100) {
    page {
      id
      originalFilename
      state
      processedAt
    }
  }
}

# Response: only invoices with ID > 100
```

### Typical Sync Pattern

```javascript
// Pseudocode for incremental sync
let lastSyncId = loadFromDatabase(); // e.g., 1234

// Fetch new invoices
const result = await graphql(`
  query {
    invoices(limit: 100, sinceId: ${lastSyncId}) {
      page {
        id
        originalFilename
        state
      }
    }
  }
`);

// Process invoices
for (const invoice of result.data.invoices.page) {
  processInvoice(invoice);
  lastSyncId = Math.max(lastSyncId, invoice.id);
}

// Save new lastSyncId
saveToDatabase(lastSyncId);
```

## Combining sinceId and Pagination

You can use both `sinceId` and `cursor` together for large incremental syncs:

```graphql
query {
  invoices(limit: 50, sinceId: 1000, cursor: "eyJpZCI6MTA1MH0=") {
    page {
      id
      originalFilename
    }
    cursor
  }
}
```

This fetches invoices with ID > 1000, starting from the position indicated by the cursor.

## Invoice States

Invoices are returned in one of three final states:

- **OK** - Successfully processed
- **REJECTED** - Processing failed (check `errorReason` field)
- **ERROR** - System error during processing (check `errorReason` field)

Intermediate processing states (e.g., UPLOADED, PROCESSING) are not exposed via the API.

## Error Handling

### GraphQL Errors

GraphQL errors are returned in the response with HTTP 200:

```json
{
  "errors": [
    {
      "message": "Limit cannot exceed 100",
      "path": ["invoices"],
      "extensions": {
        "code": "BAD_USER_INPUT"
      }
    }
  ]
}
```

### HTTP Errors

Authentication and system errors return appropriate HTTP status codes:

| Status | Description | Action |
|--------|-------------|--------|
| 401 | Unauthorized - invalid/expired token | Request new token |
| 429 | Too Many Requests - rate limited | Wait and retry |
| 500 | Internal Server Error | Contact support |

## Best Practices

### 1. Request Only Needed Fields

GraphQL allows you to request exactly the fields you need:

```graphql
# Good - minimal query
query {
  invoices(limit: 50) {
    page {
      id
      invoiceReference
      totalGrossAmount
    }
  }
}
```

### 2. Use Appropriate Page Sizes

Balance between number of requests and response size:

- Small syncs: 20-50 items
- Bulk downloads: 50-100 items
- Don't exceed: 100 items (enforced by API)

### 3. Handle Null Values

Always check for null values before processing:

```javascript
const invoice = data.invoices.page[0];

if (invoice.totalGrossAmount) {
  const amount = parseFloat(invoice.totalGrossAmount);
  // Process amount
}
```

### 4. Use Introspection During Development

Let your GraphQL client tools use introspection to provide:
- Autocomplete for field names
- Type validation
- Documentation hints
- Error detection before sending queries

## Next Steps

- [Testing with Bruno](testing-with-bruno.md) - Test queries interactively with introspection
- [Authentication](authentication.md) - Understand security model
- [Getting Started](getting-started.md) - Initial setup guide
