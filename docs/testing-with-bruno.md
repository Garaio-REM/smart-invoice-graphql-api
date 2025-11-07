# Testing with Bruno

This guide shows you how to test the Smart.Invoice GraphQL API using [Bruno](https://www.usebruno.com/), a modern API client.

## Why Bruno?

- **Open Source**: Free and privacy-focused
- **Git-Friendly**: Collections stored as files, perfect for version control
- **Fast**: Lightweight and performant
- **GraphQL Support**: Built-in GraphQL support with syntax highlighting
- **Environment Variables**: Manage multiple environments (dev, staging, production)

## Installation

### Download Bruno

Visit [usebruno.com](https://www.usebruno.com/) and download for your platform:

- **macOS**: Download .dmg
- **Windows**: Download .exe installer
- **Linux**: Download AppImage or use package manager

### Alternative: Using Homebrew (macOS)

```bash
brew install bruno
```

## Setting Up Your First Collection

### 1. Create a New Collection

1. Open Bruno
2. Click "Create Collection"
3. Name it "Smart.Invoice API"
4. Choose a location to save (recommended: in your project folder)

### 2. Configure Environment

Bruno environments let you switch between different API instances easily.

**Create environment:**

1. Click on collection settings (gear icon)
2. Go to "Environments" tab
3. Click "Add Environment"
4. Name it "Production" (or "Development")

**Add environment variables:**

```javascript
{
  "api_url": "https://your-instance.smart-invoice.ch",
  "api_key": "your-api-key-here",
  "access_token": ""
}
```

**Variables explained:**

- `api_url`: Your Smart.Invoice instance URL
- `api_key`: Your API key (from administrator)
- `access_token`: Will be set automatically by authentication request

### 3. Create Token Request

Create your first request to get an access token:

**Request Details:**

- **Name**: "Get Access Token"
- **Method**: POST
- **URL**: `{{api_url}}/api/token`
- **Body Type**: JSON

**Request Body:**

```json
{
  "grant_type": "app_key",
  "app_key": "{{api_key}}"
}
```

**Post-Response Script:**

Add this script to automatically save the token to your environment:

```javascript
// Parse the response
const data = res.getBody();

// Save access token to environment
if (data && data.access_token) {
  bru.setEnvVar("access_token", data.access_token);
  console.log("Access token saved to environment");
} else {
  console.error("No access token in response");
}
```

**Run the request:**

1. Click "Send"
2. Verify you get a successful response with `access_token`
3. Check that `access_token` is now set in your environment variables

### 4. Create GraphQL Request

Now create a request to query invoices:

**Request Details:**

- **Name**: "Query Invoices"
- **Method**: POST
- **URL**: `{{api_url}}/api/graphql`
- **Body Type**: GraphQL

**Headers:**

```
Authorization: Bearer {{access_token}}
```

**GraphQL Query:**

```graphql
query GetInvoices($limit: Int, $cursor: String) {
  invoices(limit: $limit, cursor: $cursor) {
    page {
      id
      originalFilename
      state
      processedAt
      invoiceReference
      totalGrossAmount
    }
    cursor
  }
}
```

**Query Variables:**

```json
{
  "limit": 10,
  "cursor": null
}
```

**Run the request:**

1. Make sure you've run "Get Access Token" first
2. Click "Send"
3. View the invoice data in the response

## Complete Collection Structure

Here's a recommended collection structure:

```
Smart.Invoice API/
├── Authentication/
│   └── Get Access Token
├── Basic Queries/
│   ├── Query Invoices (Simple)
│   ├── Query Invoices (Full Fields)
│   └── Query Single Invoice
├── Pagination/
│   ├── First Page
│   ├── Next Page (with cursor)
│   └── Large Dataset (100 items)
└── Incremental Sync/
    ├── Initial Sync
    └── Delta Sync (since ID)
```

## Example Requests

### Request 1: Get Access Token

**Method**: POST  
**URL**: `{{api_url}}/api/token`

**Body:**

```json
{
  "grant_type": "app_key",
  "app_key": "{{api_key}}"
}
```

**Tests:**

```javascript
test("Status is 200", () => {
  expect(res.getStatus()).to.equal(200);
});

test("Has access token", () => {
  const data = res.getBody();
  expect(data.access_token).to.be.a('string');
  expect(data.access_token.length).to.be.greaterThan(0);
});
```

### Request 2: Simple Invoice Query

**Method**: POST  
**URL**: `{{api_url}}/api/graphql`  
**Headers**: `Authorization: Bearer {{access_token}}`

**GraphQL Body:**

```graphql
query {
  invoices(limit: 5) {
    page {
      id
      originalFilename
      state
    }
  }
}
```

**Tests:**

```javascript
test("Status is 200", () => {
  expect(res.getStatus()).to.equal(200);
});

test("Has invoices data", () => {
  const data = res.getBody();
  expect(data.data.invoices).to.be.an('object');
  expect(data.data.invoices.page).to.be.an('array');
});

test("Invoices have required fields", () => {
  const invoices = res.getBody().data.invoices.page;
  if (invoices.length > 0) {
    const invoice = invoices[0];
    expect(invoice).to.have.property('id');
    expect(invoice).to.have.property('originalFilename');
    expect(invoice).to.have.property('state');
  }
});
```

### Request 3: Full Invoice Details

**GraphQL Body:**

```graphql
query GetInvoiceDetails($limit: Int!) {
  invoices(limit: $limit) {
    page {
      id
      originalFilename
      state
      processedAt
      errorReason
      accountingReference
      masterdataReference
      creditorReference
      iban
      orderReference
      languageCode
      invoiceDate
      dueDate
      externalReference
      qrReference
      contractReference
      totalGrossAmount
      invoiceReference
      servicePeriod {
        begin
        end
      }
      paymentSlipNote
      discountPercent
      discountDays
      documentUrl
    }
    cursor
  }
}
```

**Variables:**

```json
{
  "limit": 10
}
```

### Request 4: Pagination Example

**GraphQL Body:**

```graphql
query GetPage($limit: Int!, $cursor: String) {
  invoices(limit: $limit, cursor: $cursor) {
    page {
      id
      originalFilename
      state
    }
    cursor
  }
}
```

**Variables (First Page):**

```json
{
  "limit": 20,
  "cursor": null
}
```

**Variables (Next Page):**

```json
{
  "limit": 20,
  "cursor": "eyJpZCI6MjB9"
}
```

**Post-Response Script (Auto-pagination):**

```javascript
const data = res.getBody();

if (data && data.data && data.data.invoices) {
  const cursor = data.data.invoices.cursor;
  
  if (cursor) {
    // Save cursor for next page
    bru.setEnvVar("next_cursor", cursor);
    console.log(`Next cursor saved: ${cursor}`);
  } else {
    console.log("No more pages");
    bru.setEnvVar("next_cursor", null);
  }
  
  const count = data.data.invoices.page.length;
  console.log(`Fetched ${count} invoices`);
}
```

### Request 5: Incremental Sync

**GraphQL Body:**

```graphql
query IncrementalSync($sinceId: Int, $limit: Int!) {
  invoices(sinceId: $sinceId, limit: $limit) {
    page {
      id
      originalFilename
      state
      processedAt
      invoiceReference
    }
  }
}
```

**Variables (Initial Sync):**

```json
{
  "limit": 100,
  "sinceId": null
}
```

**Variables (Delta Sync):**

```json
{
  "limit": 100,
  "sinceId": 1234
}
```

**Post-Response Script:**

```javascript
const data = res.getBody();

if (data && data.data && data.data.invoices) {
  const invoices = data.data.invoices.page;
  
  if (invoices.length > 0) {
    // Find highest ID
    const maxId = Math.max(...invoices.map(inv => inv.id));
    bru.setEnvVar("last_sync_id", maxId);
    console.log(`Last sync ID updated to: ${maxId}`);
    console.log(`Fetched ${invoices.length} new invoices`);
  } else {
    console.log("No new invoices");
  }
}
```

## Advanced Features

### Using Pre-Request Scripts

Run code before each request:

```javascript
// Check if token is about to expire
const tokenSetAt = bru.getEnvVar("token_set_at");
const now = Date.now();

// Token expires after 1 hour
if (!tokenSetAt || (now - tokenSetAt) > 55 * 60 * 1000) {
  console.log("Token expired or expiring soon, please refresh");
  // You could automatically call the token endpoint here
}
```

### Collection-Level Scripts

Set scripts that run for all requests in the collection:

**Pre-Request (Collection Level):**

```javascript
// Log request details
console.log(`Making request to: ${req.getUrl()}`);
console.log(`Method: ${req.getMethod()}`);
```

**Post-Response (Collection Level):**

```javascript
// Log response time
console.log(`Response time: ${res.getResponseTime()}ms`);

// Check for errors
if (res.getStatus() >= 400) {
  console.error(`Error response: ${res.getStatus()}`);
}
```

### Environment Switching

Create multiple environments for different stages:

**Development:**

```json
{
  "api_url": "http://localhost:4000",
  "api_key": "dev-api-key-here",
  "access_token": ""
}
```

**Staging:**

```json
{
  "api_url": "https://staging.smart-invoice.ch",
  "api_key": "staging-api-key-here",
  "access_token": ""
}
```

**Production:**

```json
{
  "api_url": "https://production.smart-invoice.ch",
  "api_key": "prod-api-key-here",
  "access_token": ""
}
```

Switch between them using the environment dropdown in Bruno.

## Troubleshooting

### "Unauthorized" Error on GraphQL Request

**Problem:** Getting 401 Unauthorized

**Solutions:**

1. Check that you ran "Get Access Token" first
2. Verify `{{access_token}}` is set in environment
3. Token may have expired - re-run "Get Access Token"
4. Check Authorization header is set correctly

### "Invalid API key" Error

**Problem:** Token request returns error

**Solutions:**

1. Verify `{{api_key}}` is set correctly in environment
2. Check for extra spaces or newlines in API key
3. Confirm API key is still active with administrator

### GraphQL Syntax Errors

**Problem:** GraphQL query fails to parse

**Solutions:**

1. Use Bruno's GraphQL mode for syntax highlighting
2. Validate query structure (opening/closing braces)
3. Check field names match the schema exactly
4. Test in GraphQL Playground first if available

### Environment Variables Not Updating

**Problem:** Variables don't save

**Solutions:**

1. Make sure you're using `bru.setEnvVar()` not regular JavaScript variables
2. Check you've selected the correct environment
3. Save the collection after making changes
4. Restart Bruno if variables don't persist

## Best Practices

### 1. Use Environment Variables

Never hardcode values - use variables:

```
❌ Bad: "https://production.smart-invoice.ch/api/token"
✅ Good: "{{api_url}}/api/token"
```

### 2. Add Tests to Requests

Verify responses automatically:

```javascript
test("Invoice has required fields", () => {
  const invoice = res.getBody().data.invoices.page[0];
  expect(invoice.id).to.be.a('number');
  expect(invoice.state).to.be.oneOf(['OK', 'REJECTED', 'ERROR']);
});
```

### 3. Use Descriptive Names

Name requests clearly:

```
❌ Bad: "Request 1", "Test", "GraphQL"
✅ Good: "Get Access Token", "Query Recent Invoices", "Paginate All Invoices"
```

### 4. Organize with Folders

Group related requests:

```
Authentication/
Queries/
  Basic/
  Advanced/
Pagination/
Error Scenarios/
```

### 5. Document Your Requests

Add documentation to each request explaining:

- What the request does
- Required variables
- Expected response
- Common issues

## Exporting and Sharing

### Export Collection

Share your Bruno collection with team members:

1. Collections are stored as files in the folder you chose
2. Commit to Git for version control
3. Team members clone and open in Bruno
4. Each person sets their own environment variables

### Gitignore Sensitive Data

Create a `.gitignore` in your collection folder:

```gitignore
# Bruno environment files (contain API keys)
environments/*.bru

# Keep the example environment
!environments/example.bru
```

### Example Environment Template

Create `environments/example.bru`:

```
vars {
  api_url: https://smart-invoice.fly.dev
  api_key: your-api-key-here
  access_token: 
}
```

Team members copy this to create their own environment.

## Next Steps

- [GraphQL API Reference](graphql-api.md) - Explore schema with introspection
- [Authentication](authentication.md) - Understand token management
- [Getting Started](getting-started.md) - Initial setup guide

## Resources

- [Bruno Documentation](https://docs.usebruno.com/)
- [GraphQL Documentation](https://graphql.org/learn/)
- [Smart.Invoice API Reference](graphql-api.md)
