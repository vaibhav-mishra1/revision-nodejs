# Question 04 — HTTP Methods, Headers, and Request Bodies

## 📋 What Was Asked

Extend the HTTP server to handle POST requests with JSON bodies. Parse request bodies without Express middleware. Validate fields, check content types, and return proper status codes.

---

## 🏗️ Architecture & File Structure

```
question-04-http-methods-requests/
├── server.js                        ← Entry point, top-level error handler
├── routes/
│   └── userRoutes.js                ← Route matching and dispatch
├── controllers/
│   └── userController.js            ← Request-level logic (parse, validate, respond)
├── services/
│   └── userService.js               ← In-memory data management
├── validators/
│   └── userValidator.js             ← Input validation rules
├── utils/
│   ├── readRequestBody.js           ← Manual body parsing (replaces body-parser)
│   └── sendJSON.js                  ← Response helper
└── data/
    └── users.js                     ← In-memory user array
```

### Why This MVC-Like Structure?

This is the first question where we break into a layered architecture:

```
Request → Route → Controller → Validator → Service → Response
```

| Layer | Responsibility | Why Separate? |
|-------|---------------|---------------|
| **Routes** | Match URL + method | Routing logic stays simple and scannable |
| **Controllers** | Orchestrate — parse body, validate, call service, send response | Glue code between HTTP and business logic |
| **Validators** | Check input data | Reusable; changes to validation rules don't affect other layers |
| **Services** | Data operations | Can be tested independently without HTTP concerns |
| **Utils** | Generic helpers | Reused across the project |

---

## 🔑 Core Concepts You MUST Know

### 1. Parsing the Request Body — THE Key Concept

This is what Express's `body-parser` does automatically. Here we do it manually:

```js
function readRequestBody(req) {
    return new Promise((resolve, reject) => {
        let body = '';
        let tooLarge = false;

        req.on('data', (chunk) => {        // Each chunk of data
            if (tooLarge) return;
            body += chunk;
            if (Buffer.byteLength(body) > MAX_BODY_SIZE) {
                tooLarge = true;
                const error = new Error('Request body is too large');
                error.statusCode = 413;
                reject(error);
            }
        });

        req.on('end', () => {              // All chunks received
            if (tooLarge) return;
            try {
                resolve(JSON.parse(body));
            } catch {
                const error = new Error('Invalid JSON');
                error.statusCode = 400;
                reject(error);
            }
        });

        req.on('error', reject);           // Stream error
    });
}
```

**Cross-Questions:**

- **Q: Why is `req` a stream?**
  - HTTP request bodies can be of any size. Node reads them as a stream to avoid loading a 1GB file upload into memory at once. Each `data` event gives you a chunk (typically 64KB).

- **Q: Why wrap in a Promise?**
  - The `data`/`end`/`error` events are Node's stream API (event-driven). Wrapping in a Promise lets us use `await readRequestBody(req)` — much cleaner.

- **Q: Why check `Buffer.byteLength()` and not `body.length`?**
  - `body.length` gives character count. `Buffer.byteLength()` gives byte count. Unicode characters (emojis, Chinese, etc.) can be multiple bytes per character. Security-wise, you want byte limits.

- **Q: What's the 1MB limit for?**
  - Prevents denial-of-service attacks. Without a limit, an attacker could send a 10GB body and exhaust server memory.

- **Q: What if the body is empty?**
  - `JSON.parse('')` throws a SyntaxError → caught and returned as 400 Bad Request.

### 2. Content-Type Validation

```js
const contentType = req.headers['content-type'];
if (contentType !== 'application/json') {
    return sendJSON(res, 415, {
        success: false,
        error: 'Unsupported Media Type',
        message: 'Content-Type must be application/json'
    });
}
```

**Why check Content-Type?**
- If someone sends `Content-Type: text/plain` or `multipart/form-data`, our JSON parser will fail.
- 415 (Unsupported Media Type) is the correct status code — tells the client exactly what's wrong.

**Cross-Question:** *What about `application/json; charset=utf-8`?*
- Our simple check `!== 'application/json'` would reject this valid Content-Type. A more robust check uses: `contentType?.includes('application/json')` or a regex. (We fix this in Q10.)

### 3. Input Validation

```js
function validateUser(data) {
    if (data === null || typeof data !== 'object' || Array.isArray(data)) {
        return 'Request body must be a valid JSON object';
    }
    if (typeof data.name !== 'string' || !data.name.trim()) {
        return 'Name is required and must be a non-empty string';
    }
    if (typeof data.email !== 'string' || 
        !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(data.email)) {
        return 'A valid email address is required';
    }
    return null;  // null = valid
}
```

**Why these specific checks?**

| Check | Why |
|-------|-----|
| `data === null` | `typeof null === 'object'` in JavaScript (historic bug) |
| `Array.isArray(data)` | Arrays are also `typeof 'object'` |
| `!data.name.trim()` | Catches `""`, `"  "` (whitespace-only strings) |
| Email regex | Basic format check: `something@something.something` |

**Cross-Question:** *Is the email regex production-ready?*
- No. Email validation is notoriously complex (see RFC 5322). For production, use a library or validate by sending a confirmation email. Our regex catches obvious junk but allows some invalid addresses.

### 4. HTTP Status Codes Deep Dive

| Code | Name | When Used in This Question |
|------|------|---------------------------|
| **200** | OK | Successful GET |
| **201** | Created | Successful POST (new resource created) |
| **400** | Bad Request | Invalid JSON body, missing fields |
| **405** | Method Not Allowed | Known route, wrong method |
| **413** | Content Too Large | Body exceeds 1MB limit |
| **415** | Unsupported Media Type | Wrong Content-Type header |
| **500** | Internal Server Error | Unexpected crashes |

**Cross-Question:** *Why 201 instead of 200 for POST?*
- HTTP semantics: 200 means "request succeeded." 201 specifically means "a new resource was created." It's more precise and tells the client they should expect a new resource in the response.

### 5. Promise-Based Error Handling in Server

```js
const server = http.createServer((req, res) => {
    Promise.resolve()
        .then(() => handleRoutes(req, res))
        .catch(error => {
            if (!res.headersSent && !res.destroyed) {
                res.writeHead(500, { 'Content-Type': 'application/json; charset=utf-8' });
                res.end(JSON.stringify({ success: false, error: 'Internal Server Error' }));
            }
        });
});
```

**Why `Promise.resolve().then()`?**
- `handleRoutes` is async. If we just call it directly, any thrown error won't be caught by the `http.createServer` callback (it doesn't support async/await natively).
- Wrapping in a Promise chain lets us `.catch()` both sync throws and rejected promises.

**Why check `res.headersSent`?**
- If a handler already started writing a response (via `res.writeHead` or `res.write`), you can't set headers again. Trying to would throw an error. So we check first.

### 6. Reading Request Headers

```js
console.log(
    `${req.method} ${req.url} | ` +
    `Content-Type: ${req.headers['content-type'] || 'not provided'} | ` +
    `Accept: ${req.headers.accept || 'not provided'}`
);
```

**Key points about headers:**
- All header names in `req.headers` are **lowercase** (Node normalizes them).
- `req.headers['content-type']` not `req.headers['Content-Type']`.
- The `Accept` header tells you what format the client wants (useful for content negotiation).

---

## 💡 The `sendJSON` Helper — Extra Headers Support

```js
function sendJSON(res, statusCode, data, extraHeaders = {}) {
    res.writeHead(statusCode, {
        'Content-Type': 'application/json; charset=utf-8',
        ...extraHeaders         // e.g., { Allow: 'GET, POST' }
    });
    res.end(JSON.stringify(data));
}
```

The `extraHeaders` parameter lets routes add specific headers (like `Allow` for 405 responses) without modifying the helper.

---

## 🎯 Cross-Questions to Prepare For

| Question | Your Answer |
|----------|------------|
| What does Express's `body-parser` do? | Exactly what our `readRequestBody` does — listens to `data`/`end` events, concatenates chunks, parses JSON. We built it from scratch. |
| Why is `req` a readable stream? | HTTP bodies can be large. Streaming processes them in chunks instead of loading everything into memory. |
| What happens if a client sends no body? | `JSON.parse('')` throws → we return 400 Bad Request with "Invalid JSON" message. |
| How would you support other Content-Types? | Check the `Content-Type` header and use different parsers: `querystring.parse()` for `application/x-www-form-urlencoded`, a multipart parser for file uploads, etc. |
| What's the `Allow` header? | When returning 405, the `Allow` header tells the client which methods are valid for that endpoint. It's required by the HTTP spec (RFC 7231). |
| Why return the created user in the response? | So the client gets the server-generated fields (`id`, timestamps) without making another request. Standard REST practice. |
| What if two POSTs happen simultaneously? | In this in-memory implementation, Node's single-threaded nature prevents true race conditions. But with file-based storage, concurrent writes could conflict. |

---

## 🔧 How to Run

```bash
cd question-04-http-methods-requests
node server.js
```

## 🧪 Test with curl

```bash
# GET all users
curl http://localhost:3000/users

# POST — create user (valid)
curl -X POST http://localhost:3000/users \
  -H "Content-Type: application/json" \
  -d '{"name": "Alice", "email": "alice@example.com"}'

# POST — missing Content-Type (415)
curl -X POST http://localhost:3000/users \
  -d '{"name": "Alice"}'

# POST — invalid JSON (400)
curl -X POST http://localhost:3000/users \
  -H "Content-Type: application/json" \
  -d '{invalid}'

# POST — missing fields (400)
curl -X POST http://localhost:3000/users \
  -H "Content-Type: application/json" \
  -d '{"name": ""}'
```

---

## 📚 Key Node.js Concepts Demonstrated

1. **Manual Body Parsing** — Reading request stream without middleware
2. **Content-Type Checking** — Enforcing `application/json`
3. **Request Size Limiting** — DoS prevention with `Buffer.byteLength`
4. **Input Validation** — Type checks, format checks, null guards
5. **HTTP Status Codes** — 201, 400, 405, 413, 415 with correct semantics
6. **Layered Architecture** — Routes → Controllers → Validators → Services
7. **Promise-Based Error Catching** — Top-level catch for async route handlers
