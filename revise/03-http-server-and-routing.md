# Question 03 — HTTP Server and Request Routing

## 📋 What Was Asked

Build an HTTP server using Node.js's built-in `http` module with routes for `GET /`, `GET /users`, `GET /health`, 404 handling for unknown routes, and proper JSON responses.

---

## 🏗️ File Structure

```
question-03-http-server/
├── server.js    ← Single file — server + routing + response handling
```

This is intentionally simple. The assignment builds complexity progressively — Q4 adds request bodies, Q5 adds CRUD, etc.

---

## 🔑 Core Concepts You MUST Know

### 1. The `http.createServer()` Function

```js
const http = require('http');

const server = http.createServer((req, res) => {
    // This callback fires for EVERY incoming request
    // req = IncomingMessage (readable stream)
    // res = ServerResponse (writable stream)
});
```

**Cross-Question:**
- **Q: What are `req` and `res` exactly?**
  - `req` is an instance of `http.IncomingMessage` — it's a **readable stream** containing the request data (headers, method, URL, body).
  - `res` is an instance of `http.ServerResponse` — it's a **writable stream** where you write headers and body data back to the client.

- **Q: Why not use Express?**
  - The assignment explicitly prohibits Express. This demonstrates understanding of what Express abstracts away. Every Express `app.get()` is just this pattern under the hood.

### 2. URL Parsing

```js
const pathname = new URL(
    req.url,
    `http://${req.headers.host || 'localhost'}`
).pathname;
```

**Why `new URL()` instead of just `req.url`?**
- `req.url` gives you `/users?page=1` — the raw path + query string.
- `new URL()` parses it properly, giving you `.pathname` (`/users`), `.searchParams` (query parameters), etc.
- The second argument is the base URL — needed because `req.url` is a relative path.

**Cross-Question:** *Why `req.headers.host || 'localhost'`?*
- The `Host` header could theoretically be missing. The fallback prevents `new URL()` from throwing.

### 3. Routing Pattern — Manual `if` Matching

```js
if (method === 'GET' && pathname === '/') { ... }
if (method === 'GET' && pathname === '/users') { ... }
if (method === 'GET' && pathname === '/health') { ... }
```

**Why simple `if` statements?**
- For 3 routes, a router library is overkill.
- Shows you understand that routing is just "matching method + path and calling the right function."
- Express's `app.get('/users', handler)` does exactly this internally.

**Cross-Question:** *How would you scale this?*
- For more routes: create a route table (Map or array of `{ method, path, handler }`).
- For dynamic segments (`/users/:id`): use regex matching (which we do in Q5).

### 4. JSON Response Helper

```js
function sendJSON(res, statusCode, data) {
    res.writeHead(statusCode, {
        'Content-Type': 'application/json; charset=utf-8'
    });
    res.end(JSON.stringify(data));
}
```

**Why a helper function?**
- DRY principle — every response needs `Content-Type` and `JSON.stringify`.
- Ensures consistency — all responses have the same structure.
- Prevents bugs — forgetting `Content-Type` would cause clients to not parse JSON correctly.

**Cross-Questions:**
- **Q: Why `charset=utf-8`?**
  - JSON is defined as UTF-8 by RFC 8259. Including it explicitly ensures older clients handle encoding correctly.

- **Q: Why `res.writeHead()` and not `res.setHeader()`?**
  - `writeHead()` sets the status code AND headers in one call, then flushes the headers. `setHeader()` just buffers a single header. Both work, but `writeHead()` is more explicit.

- **Q: Why `res.end()` and not `res.write()`?**
  - `res.end()` writes data AND signals the response is complete. `res.write()` writes data but keeps the connection open for more writes. For simple JSON responses, `res.end()` is cleaner.

### 5. HTTP Status Codes Used

| Code | Meaning | When Used |
|------|---------|-----------|
| **200** | OK | Successful GET requests |
| **404** | Not Found | Unknown routes |
| **405** | Method Not Allowed | Known route, wrong method (e.g., POST to `/health`) |

**Cross-Question:** *Why 405 and not 404 for wrong methods?*
- 404 means "this resource doesn't exist." 405 means "this resource exists, but you're using the wrong HTTP method." It's more specific and helpful for API consumers.
- We also set the `Allow` header (`Allow: GET`) to tell the client which methods ARE supported.

### 6. Error Event Handling

```js
server.on('error', (err) => {
    console.error('Server error:', err.message);
    process.exitCode = 1;
});
```

**Why `process.exitCode` instead of `process.exit(1)`?**
- `process.exit(1)` terminates immediately — pending operations (file writes, active connections) are killed.
- `process.exitCode = 1` sets the exit code but lets Node exit gracefully after pending operations complete.

---

## 💡 Response Structure Design

Every response follows a consistent envelope:

```json
{
    "success": true,          // Boolean flag for quick client-side checks
    "message": "...",         // Human-readable message
    "data": { ... },          // Payload (only on success)
    "error": "Not Found"      // Error type (only on failure)
}
```

**Why a consistent envelope?**
- Clients can always check `response.success` without guessing.
- Makes error handling uniform across all endpoints.
- Standard practice in production APIs (similar to JSend specification).

---

## 🎯 Cross-Questions to Prepare For

| Question | Your Answer |
|----------|------------|
| How does `http.createServer` work internally? | It creates a TCP server, parses incoming HTTP requests, and invokes your callback with `req` and `res` objects. |
| What's the difference between `http` and `https`? | `https` wraps `http` with TLS encryption. You need an SSL certificate. Code is almost identical, just `https.createServer({key, cert}, handler)`. |
| Why is the response a stream? | Allows you to send data incrementally (e.g., large files). `res.write()` can be called multiple times before `res.end()`. |
| What happens if you never call `res.end()`? | The client connection hangs until it times out. The server also leaks resources. |
| How would you add logging? | Use a middleware-like pattern: log `req.method`, `req.url`, timestamp at the start, and response status on `res.on('finish', ...)`. (We do this in Q10.) |
| What is `req.headers.host`? | The `Host` header sent by the client. Contains the domain and port (e.g., `localhost:3000`). Required by HTTP/1.1 spec. |
| How does Node handle multiple requests if it's single-threaded? | The event loop. Each request handler is async — while one request waits for I/O (database, file), Node processes other requests. |

---

## 🔧 How to Run

```bash
cd question-03-http-server
node server.js
```

## 🧪 Test with curl

```bash
# GET /
curl http://localhost:3000/

# GET /users
curl http://localhost:3000/users

# GET /health
curl http://localhost:3000/health

# 404 — unknown route
curl http://localhost:3000/unknown

# 405 — wrong method
curl -X POST http://localhost:3000/health
```

---

## 📚 Key Node.js Concepts Demonstrated

1. **`http` Core Module** — Creating servers without frameworks
2. **Request/Response Cycle** — `req` (readable stream), `res` (writable stream)
3. **Manual Routing** — Method + pathname matching
4. **URL Parsing** — `new URL()` API
5. **HTTP Status Codes** — 200, 404, 405 with correct semantics
6. **JSON Responses** — Proper `Content-Type`, consistent structure
7. **Server Error Handling** — `server.on('error')` for startup failures
