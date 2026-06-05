---
name: filemaker-connect
description: Connect to FileMaker Server Data API. Use when building apps that need to fetch data from FileMaker databases, or when implementing FileMaker authentication.
disable-model-invocation: false
user-invocable: true
---

# FileMaker Server Data API Connection Guide

This skill provides knowledge on how to connect to FileMaker Server's Data API from web applications.

## Overview

FileMaker Server exposes a REST API at `/fmi/data/vLatest/` that requires:
1. Session-based authentication (Basic Auth → Bearer token)
2. A serverless proxy to avoid CORS issues (FileMaker doesn't support CORS headers)

## Architecture

```
Browser → Serverless Function (Vercel/Netlify) → FileMaker Server
              ↓
         Handles CORS + Auth
```

## Authentication Flow

1. **Login**: POST to `/fmi/data/vLatest/databases/{database}/sessions` with Basic Auth
2. **Use token**: All subsequent requests use Bearer token
3. **Logout**: DELETE to `/fmi/data/vLatest/databases/{database}/sessions/{token}`

## Key Endpoints

| Action | Method | URL |
|--------|--------|-----|
| Login | POST | `/fmi/data/vLatest/databases/{db}/sessions` |
| Logout | DELETE | `/fmi/data/vLatest/databases/{db}/sessions/{token}` |
| Get records | GET | `/fmi/data/vLatest/databases/{db}/layouts/{layout}/records` |
| Find records | POST | `/fmi/data/vLatest/databases/{db}/layouts/{layout}/_find` |
| Run script | GET | `/fmi/data/vLatest/databases/{db}/layouts/{layout}/script/{scriptName}` |

## Serverless Proxy Template (Vercel)

Create `api/filemaker.js`:

```javascript
async function authenticate(serverUrl, database, username, password) {
  const baseUrl = serverUrl.replace(/\/+$/, '')
  const url = `${baseUrl}/fmi/data/vLatest/databases/${encodeURIComponent(database)}/sessions`
  const credentials = Buffer.from(`${username}:${password}`).toString('base64')

  const response = await fetch(url, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Basic ${credentials}`
    },
    body: JSON.stringify({})
  })

  if (!response.ok) {
    if (response.status === 401) throw new Error('Invalid credentials')
    if (response.status === 404) throw new Error(`Database '${database}' not found`)
    throw new Error(`Authentication failed: ${response.status}`)
  }

  const data = await response.json()
  if (data.messages?.[0]?.code !== '0') {
    throw new Error(data.messages?.[0]?.message || 'Authentication failed')
  }

  return data.response.token
}

async function fetchLayout(token, serverUrl, database, layoutName) {
  const baseUrl = serverUrl.replace(/\/+$/, '')
  const url = `${baseUrl}/fmi/data/vLatest/databases/${encodeURIComponent(database)}/layouts/${encodeURIComponent(layoutName)}/records?_limit=10000`

  const response = await fetch(url, {
    method: 'GET',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  })

  if (!response.ok) {
    throw new Error(`Failed to fetch layout '${layoutName}': ${response.status}`)
  }

  const data = await response.json()
  return data.response.data || []
}

async function logout(token, serverUrl, database) {
  try {
    const baseUrl = serverUrl.replace(/\/+$/, '')
    const url = `${baseUrl}/fmi/data/vLatest/databases/${encodeURIComponent(database)}/sessions/${token}`
    await fetch(url, { method: 'DELETE' })
  } catch {
    // Ignore logout errors
  }
}

export default async function handler(req, res) {
  res.setHeader('Access-Control-Allow-Origin', '*')
  res.setHeader('Access-Control-Allow-Methods', 'POST, OPTIONS')
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type')

  if (req.method === 'OPTIONS') return res.status(200).end()
  if (req.method !== 'POST') return res.status(405).json({ error: 'Method not allowed' })

  const { action, serverUrl, database, username, password, layout } = req.body

  if (!serverUrl || !database || !username || !password) {
    return res.status(400).json({ error: 'Missing credentials' })
  }

  try {
    const token = await authenticate(serverUrl, database, username, password)

    try {
      if (action === 'test') {
        return res.status(200).json({ success: true })
      }

      if (action === 'fetch' && layout) {
        const records = await fetchLayout(token, serverUrl, database, layout)
        return res.status(200).json({ records })
      }

      return res.status(400).json({ error: 'Invalid action' })
    } finally {
      await logout(token, serverUrl, database)
    }
  } catch (error) {
    return res.status(500).json({ error: error.message })
  }
}
```

## Frontend Client

```javascript
const API_ENDPOINT = '/api/filemaker'

export async function testConnection(credentials) {
  const response = await fetch(API_ENDPOINT, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ action: 'test', ...credentials })
  })
  if (!response.ok) {
    const data = await response.json()
    throw new Error(data.error || 'Connection failed')
  }
  return true
}

export async function fetchLayout(credentials, layout) {
  const response = await fetch(API_ENDPOINT, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ action: 'fetch', layout, ...credentials })
  })
  const data = await response.json()
  if (!response.ok) throw new Error(data.error)
  return data.records
}
```

## FileMaker Record Structure

Records from the API have this shape:
```javascript
{
  recordId: "123",
  modId: "1",
  fieldData: {
    fieldName: "value",
    // ... all fields from the layout
  }
}
```

Transform to flat objects:
```javascript
function transformRecord(record) {
  return {
    ...record.fieldData,
    _recordId: record.recordId,
    _modId: record.modId
  }
}
```

## Common Issues

1. **CORS errors**: You MUST use a serverless proxy. FileMaker Server doesn't support CORS.
2. **Self-signed SSL**: Use `NODE_TLS_REJECT_UNAUTHORIZED=0` in dev (not production!)
3. **Session limits**: FileMaker has limited concurrent sessions. Always logout after use.
4. **Layout not found**: Ensure the layout exists and the user has access to it.
5. **Empty results**: Check if the layout has records and proper field access.
6. **Default `_limit` is 100**: The Data API returns only 100 records unless you set
   `?_limit=10000` (or page). A "missing rows" bug is often just this.
7. **Fields must be ON the layout** *(very common gotcha)*: the Data API only returns
   fields physically placed on the layout it reads. A field that exists in the table
   but isn't on the layout comes back **missing/empty — silently**. When a value is
   unexpectedly blank, check the layout first.
8. **Error 802 "Unable to open file"** = the **database name is wrong / file not hosted
   / Data API not enabled** for it. The `{database}` path segment is the **file name
   WITHOUT `.fmp12`**, exact case. This fails at the `/sessions` step before credentials
   are checked, so 802 ≠ a password problem.

## FileMaker error codes (quick reference)

The Data API returns `{ "messages": [{ "code": "...", "message": "..." }], ... }`.
Code `"0"` = success. **An HTTP 200 can still carry an FM error — always check
`data.messages[0].code !== '0'`.** Common failures:

| Code | Meaning | Usual fix |
|------|---------|-----------|
| `802` | Unable to open file | Wrong `database` name (no `.fmp12`), file not hosted, or Data API not enabled |
| `212` (or HTTP 401) | Invalid user account/password | Check credentials + that the account has `fmrest` |
| `9` | Insufficient privileges | Privilege set needs record + layout access |
| `105` | Layout missing | Layout not present or not accessible to the account |
| `952` | Invalid/expired Data API token | Re-authenticate; tokens expire after ~15 min idle |

## Data-type pitfalls (real-world)

Everything in `fieldData` comes back as **strings in the file's locale**. Parse defensively:

- **Numbers** — a Dutch file returns `"1.234,5"` (dot thousands, comma decimal):
  ```javascript
  function parseNumber(v) {
    if (v === null || v === undefined || v === '') return 0
    if (typeof v === 'number') return v
    return parseFloat(String(v).replace(/\./g, '').replace(',', '.')) || 0
  }
  ```
- **Dates** — returned in the **file's date format, NOT ISO**. For a Dutch file that's
  `DD-MM-YYYY`. **`new Date("01-06-2026")` is WRONG** (parsed as January + timezone
  off-by-one) and **`new Date("25-08-2026")` is `Invalid Date`** → if you drop invalid
  dates you **silently lose records**. Parse explicitly, timezone-safe:
  ```javascript
  function dateToSerial(y, m, d) {            // → Excel/day serial, UTC-safe
    return Math.floor(Date.UTC(y, m - 1, d) / 86400000) + 25569
  }
  function parseFMDate(value) {               // ISO, or D/M/Y (day-first when ambiguous)
    if (value == null || value === '') return null
    if (typeof value === 'number') return value
    const s = String(value).trim()
    const iso = s.match(/^(\d{4})-(\d{1,2})-(\d{1,2})/)
    if (iso) return dateToSerial(+iso[1], +iso[2], +iso[3])
    const p = s.split(/\D+/).filter(Boolean).map(Number)
    if (p.length >= 3 && p[2] >= 1000) {
      const [a, b, y] = p
      const day = a > 12 ? a : b               // ambiguous → day-first (NL/EU)
      const month = a > 12 ? b : (b > 12 ? a : b)
      return dateToSerial(y, month, day)
    }
    return null
  }
  ```
  Confirm the file's date format with a value where the day is **> 12** so it's unambiguous.
- **Booleans / flags** (e.g. an `active` field) — come as `"1"`/`"0"`/`""`. Pick a sane
  default for empty (usually "true" so you don't hide records unexpectedly).
- **Numeric-looking keys** — an article number may come back as a **number** in
  `fieldData` but a **string** in your request. Coerce both sides when matching.
- **Robust field mapping** — field names vary by file; map several candidates:
  `data.articleNumber || data.ArticleNumber || data.artnr`.

## Server-to-server endpoints (env vars + API key)

The browser proxy stores per-user credentials. For an **unattended** endpoint (another
system — or FileMaker itself — calling your API), don't pass FileMaker credentials in
each request: keep them as **environment variables** and gate the endpoint with a
separate **API key**.

```javascript
// Vercel env: FM_SERVER_URL, FM_DATABASE, FM_USERNAME, FM_PASSWORD, MY_API_KEY
import crypto from 'crypto'
export default async function handler(req, res) {
  if (req.method !== 'POST') return res.status(405).json({ error: 'Method not allowed' })
  const key = req.headers['x-api-key']
  if (!key || !process.env.MY_API_KEY ||
      Buffer.from(key).length !== Buffer.from(process.env.MY_API_KEY).length ||
      !crypto.timingSafeEqual(Buffer.from(key), Buffer.from(process.env.MY_API_KEY))) {
    return res.status(401).json({ error: 'Unauthorized' })
  }
  const creds = {
    serverUrl: process.env.FM_SERVER_URL, database: process.env.FM_DATABASE,
    username: process.env.FM_USERNAME, password: process.env.FM_PASSWORD
  }
  // ... authenticate(creds) → fetchLayout(s) → logout, then return JSON
}
```
- **Env-var changes only apply to NEW deployments** — redeploy after editing them.
- A **`_`-prefixed file** in `api/` (e.g. `api/_fmData.js`) is treated by Vercel as a
  **shared helper, not a route** — a good home for shared auth/fetch/transform code.
- On a private, key-gated endpoint it's fine (and helpful) to return `detail: err.message`
  on 500s for diagnosis — the caller is already authenticated.

## FileMaker calling OUT to a REST API (Insert from URL)

FileMaker can be the **client** too (e.g. check availability while entering an order
line). Use **Insert from URL** with cURL options:

```
"-X POST" &
" -H " & Quote("X-API-Key: " & $apiKey) &
" -H " & Quote("Content-Type: application/json") &
" -d " & Quote(JSONSetElement("{}";
    ["articleNumber"; $article; JSONString];
    ["quantity";      $qty;     JSONNumber];
    ["date";          $date;    JSONString]
))
```
- Build the body with **`JSONSetElement`** (pick the type: `JSONString`/`JSONNumber`/`JSONBoolean`).
- Wrap each header/value in **`Quote()`** so spaces survive.
- Parse the reply with **`JSONGetElement($result; "path.to.value")`** (dot paths into
  objects, `[n]` for arrays).
- Set **"Verify SSL Certificates"** appropriately; for self-signed dev certs add
  `--insecure` to the cURL options (never in production).
- Send dates as **ISO (`YYYY-MM-DD`)** to dodge the day/month ambiguity above.

## Running Scripts

To run a FileMaker script:
```javascript
async function runScript(token, serverUrl, database, layout, scriptName, parameter = '') {
  const baseUrl = serverUrl.replace(/\/+$/, '')
  let url = `${baseUrl}/fmi/data/vLatest/databases/${encodeURIComponent(database)}/layouts/${encodeURIComponent(layout)}/script/${encodeURIComponent(scriptName)}`

  if (parameter) {
    url += `?script.param=${encodeURIComponent(parameter)}`
  }

  const response = await fetch(url, {
    method: 'GET',
    headers: { 'Authorization': `Bearer ${token}` }
  })

  const data = await response.json()
  return data.response?.scriptResult
}
```

## Required FileMaker Setup

1. Enable FileMaker Data API in FileMaker Server Admin Console
2. Create a FileMaker account with `fmrest` extended privilege
3. Create layouts for each table you want to access (API uses layouts, not tables)
4. Ensure field-level access is configured correctly

## Example: Fetching Multiple Layouts

```javascript
const token = await authenticate(serverUrl, database, username, password)
try {
  const [products, orders, customers] = await Promise.all([
    fetchLayout(token, serverUrl, database, 'products'),
    fetchLayout(token, serverUrl, database, 'orders'),
    fetchLayout(token, serverUrl, database, 'customers')
  ])
  // Process data...
} finally {
  await logout(token, serverUrl, database)
}
```
