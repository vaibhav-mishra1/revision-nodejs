# Question 10 — API Hardening, Debugging, and Error Handling

## 📋 What Was Asked

Improve the reliability and security of the Node.js API. Implement centralized error handling, input validation, prevent information leakage, add request logging, and handle edge cases.

---

## 🏗️ Architecture & File Structure

```
question-10-api-hardening/
├── server.js                      ← Entry point + global error handler
├── .env                           ← Secrets (JWT_SECRET, JWT_EXPIRES_IN)
├── .gitignore
├── routes/
│   ├── index.js                   ← Master router
│   ├── authRoutes.js              ← POST /api/auth/login
│   └── userRoutes.js              ← /api/users/* (protected CRUD)
├── controllers/
│   ├── authController.js          ← Login logic (bcrypt + JWT)
│   └── userController.js          ← User CRUD operations
├── middleware/
│   └── authMiddleware.js          ← JWT verification + RBAC
├── utils/
│   ├── errorHandler.js            ← createError() + handleError()
│   ├── httpUtils.js               ← sendJson() + readJsonBody()
│   └── requestLogger.js           ← Method/path/status/time logging
└── data/
    ├── accounts.json              ← Login credentials
    └── user.json                  ← User records
```

### Evolution from Q5/Q6 to Q10

This is the "production-ready" version. Key improvements:

| Improvement | Before (Q5/Q6) | After (Q10) |
|-------------|----------------|-------------|
| Error handling | Ad-hoc try/catch | Centralized `createError()` + `handleError()` |
| Information leakage | Stack traces could leak | Server errors return generic messages |
| Logging | `console.log` statements | Structured request logger with response time |
| Input validation | Inline checks | UUID format validation, Content-Type regex |
| API prefix | `/users` | `/api/users` (namespaced) |
| Duplicate emails | Allowed | `409 Conflict` |

---

## 🔑 Core Concepts You MUST Know

### 1. Centralized Error Handling — THE Key Pattern

This is the most important concept in Q10. Instead of handling errors in every route handler, we:

1. **Create errors** with `createError()` — standardized error objects
2. **Throw them** from any layer (routes, controllers, middleware)
3. **Catch them** in one place — `handleError()` in server.js

#### `createError()` — Error Factory

```js
function createError(message, statusCode) {
    const error = new Error(message);
    error.statusCode = statusCode;
    error.expose = Number.isInteger(statusCode)
        && statusCode >= 400
        && statusCode < 500;
    return error;
}
```

**The `expose` flag is critical:**
- `expose: true` (4xx errors) → Client error. Safe to show the message to the user.
- `expose: false` (5xx errors) → Server error. NEVER show internal details.

This is the same pattern that Express's `http-errors` library uses.

#### `handleError()` — Centralized Error Response

```js
function handleError(error, req, res) {
    if (res.headersSent || res.destroyed) {
        res.destroy();
        return;
    }

    const statusCode = error
        && error.expose === true
        && Number.isInteger(error.statusCode)
        && error.statusCode >= 400
        && error.statusCode < 500
        ? error.statusCode
        : 500;

    const message = statusCode < 500
        && error.expose === true
        && typeof error.message === 'string'
        ? error.message
        : "Internal server error";

    sendJson(res, statusCode, {
        success: false,
        error: { message }
    });
}
```

**Why double-check `expose`?**
- Defense in depth. Even if someone creates an error with `statusCode: 500` and `expose: true`, the handler treats anything ≥ 500 as unexposed. Internal server details NEVER leak.

**Why check `res.headersSent`?**
- If a response was partially sent (e.g., streaming), you can't send headers again. Attempting to would crash the server. Better to destroy the connection.

### 2. Information Leakage Prevention

**What to NEVER expose in production:**
- Stack traces
- Database queries
- File paths
- Internal error messages from libraries
- Environment variable values

**How we prevent it:**
```js
// 4xx → show actual message (client's fault)
createError("Email already exists", 409)
// → Response: { "error": { "message": "Email already exists" } }

// 5xx → generic message (our fault)
throw new Error("ECONNREFUSED: Cannot connect to database")
// → Response: { "error": { "message": "Internal server error" } }
// The actual error is only logged server-side
```

**Cross-Question:** *What if a 500 error has useful debug info?*
- Log it server-side (with request ID for correlation). Never send it to the client. In production, use structured logging (Winston, Pino) with log levels.

### 3. Request Logger — Performance Monitoring

```js
function requestLogger(req, res) {
    const startTime = process.hrtime.bigint();
    const path = req.url.split("?")[0];

    res.on("finish", () => {
        const elapsed = process.hrtime.bigint() - startTime;
        const responseTime = Number(elapsed) / 1_000_000;

        console.log(
            `${req.method} ${path} | ` +
            `Status: ${res.statusCode} | ` +
            `Time: ${responseTime.toFixed(2)} ms`
        );
    });
}
```

**Key design decisions:**

| Decision | Why |
|----------|-----|
| `process.hrtime.bigint()` | Nanosecond precision. `Date.now()` only gives millisecond precision — not accurate enough for fast API calls. |
| `res.on("finish")` | Fires AFTER the response is completely sent. We get the accurate status code and total processing time. |
| Strip query string | `req.url.split("?")[0]` prevents logging sensitive query parameters (tokens, passwords). |
| Log format: `METHOD path \| Status: code \| Time: ms` | Structured, parseable, grep-friendly. |

**Cross-Question:** *Why `process.hrtime.bigint()` and not `Date.now()`?*
- `Date.now()` is affected by system clock changes (NTP adjustments). `hrtime` uses a monotonic clock — it always moves forward, even if the system clock changes. It also has nanosecond precision vs. millisecond.

### 4. Enhanced Input Validation

#### UUID Format Validation
```js
if (matched && !/^[0-9a-f]{8}-[0-9a-f]{4}-[1-8][0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i.test(matched[1])) {
    throw createError('User ID must be a valid UUID', 400);
}
```

**Why validate UUID format?**
- Prevents meaningless database lookups for obviously invalid IDs.
- Clear error message tells the client exactly what's expected.
- Security: prevents injection of special characters into IDs.

#### Content-Type Validation (Improved)
```js
if (!/^application\/json(?:\s*;|$)/i.test(contentType)) {
    throw createError('Content-Type must be application/json', 415);
}
```

**Why regex instead of `=== 'application/json'`?**
- Accepts `application/json`, `application/json; charset=utf-8`, `Application/JSON`.
- In Q4 we had a strict equality check that would reject valid Content-Types with parameters.

#### Duplicate Email Check
```js
const emailExists = users.some(
    user => user.email.toLowerCase() === data.email.trim().toLowerCase()
);
if (emailExists) {
    throw createError("Email already exists", 409);
}
```

**Why 409 Conflict?**
- 400 means "bad format." 409 means "your request is valid, but conflicts with existing data." More semantic and helpful.

### 5. API Namespacing with `/api/` Prefix

Routes changed from `/users` to `/api/users`, `/login` to `/api/auth/login`.

**Why?**
- Clear separation between API routes and static file routes.
- Makes Nginx routing easier (`/api/*` → Node.js, everything else → static files).
- Convention followed by most production APIs.

### 6. Improved Body Parser (Buffer-Based)

```js
function readJsonBody(req) {
    return new Promise((resolve, reject) => {
        const chunks = [];
        let size = 0;
        let settled = false;

        req.on("data", (chunk) => {
            if (settled) return;
            size += chunk.length;
            if (size > 1024 * 1024) {
                settled = true;
                chunks.length = 0;    // Free memory
                reject(createError("Request body too large", 413));
                req.resume();         // Drain remaining data
                return;
            }
            chunks.push(chunk);
        });

        req.on("end", () => {
            if (settled) return;
            try {
                settled = true;
                resolve(JSON.parse(Buffer.concat(chunks, size).toString("utf8")));
            } catch {
                settled = true;
                reject(createError("Invalid JSON body", 400));
            }
        });
    });
}
```

**Improvements over Q4/Q5 version:**

| Improvement | Before | After |
|-------------|--------|-------|
| Memory | String concatenation (`body += chunk`) | `Buffer.concat(chunks)` — more efficient for binary data |
| Size tracking | `Buffer.byteLength(body)` on every chunk | `size += chunk.length` — O(1) instead of O(n) |
| Race condition | `tooLarge` flag | `settled` flag covers all terminal states |
| Cleanup | None | `chunks.length = 0` frees buffer references, `req.resume()` drains stream |
| Error creation | Manual `new Error()` with statusCode | Uses `createError()` |

### 7. Error-Throwing Middleware (vs. Return-Based)

**Before (Q6):**
```js
function authenticateToken(req, res) {
    if (!match) {
        sendJson(res, 401, { message: 'Token required' });
        return null;    // Caller must check for null
    }
}
```

**After (Q10):**
```js
function authenticateToken(req, res) {
    if (!match) {
        throw createError('Authentication token required', 401);
        // Error propagates to handleError() automatically
    }
}
```

**Why throw instead of return null?**
- No need for `if (!user) return;` checks after every middleware call.
- The centralized error handler catches everything.
- Cleaner control flow — errors propagate naturally up the call stack.

---

## 🛡️ Failure Scenarios Tested

| Scenario | Expected Response |
|----------|------------------|
| Invalid JSON body | 400 — "Invalid JSON body" |
| Missing required fields | 400 — "Name is required..." |
| Invalid UUID format | 400 — "User ID must be a valid UUID" |
| User not found | 404 — "User not found" |
| Missing auth token | 401 — "Authentication token required" |
| Expired token | 401 — "Token expired" |
| Invalid token | 401 — "Invalid token" |
| Insufficient role | 403 — "You do not have permission" |
| Wrong method | 405 — "Method not allowed" + Allow header |
| Duplicate email | 409 — "Email already exists" |
| Wrong Content-Type | 415 — "Content-Type must be application/json" |
| Body too large | 413 — "Request body too large" |
| Unknown route | 404 — "Route not found" |
| Server crash | 500 — "Internal server error" (generic) |

---

## 🎯 Cross-Questions to Prepare For

| Question | Your Answer |
|----------|------------|
| Why centralized error handling? | Single point of control. Consistent error format. No duplicated error-response code. Easy to add logging, monitoring, or alerting in one place. |
| What information should NEVER appear in API responses? | Stack traces, file paths, database queries, internal library error messages, environment variables, server version. |
| Why `process.hrtime.bigint()` for timing? | Monotonic clock (unaffected by system clock changes), nanosecond precision, perfect for performance measurement. |
| What's the `expose` flag pattern? | 4xx errors (client mistakes) are safe to expose their message. 5xx errors (server bugs) get a generic message. This prevents information leakage from unexpected errors. |
| Why check `res.headersSent`? | If headers were already sent (partial response), attempting to set new headers throws an error. We check first and destroy the connection if needed. |
| How would you add rate limiting? | Track request counts per IP in a Map with timestamps. If count exceeds limit in the time window, return 429 Too Many Requests. In production, use Nginx rate limiting or Redis. |
| What's structured logging? | Logging in a machine-parseable format (JSON). Libraries like Pino or Winston. Enables log aggregation (ELK stack, Datadog) and searching/alerting on specific fields. |
| How would you monitor this API in production? | Health check endpoint, request metrics (Prometheus), error alerting (Sentry), structured logging (ELK), uptime monitoring. |
| What's the difference between 400, 422, and 409? | 400: malformed request (bad syntax). 422: well-formed but semantically invalid (wrong field types). 409: valid request but conflicts with current state (duplicate email). |

---

## 🔧 How to Run

```bash
cd question-10-api-hardening
node server.js
```

## 🧪 Test with curl

```bash
# Login
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "admin@example.com", "password": "admin123"}'

# Invalid JSON (400)
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{invalid}'

# Wrong Content-Type (415)
curl -X POST http://localhost:3000/api/auth/login \
  -d '{"email": "admin@example.com"}'

# Invalid UUID (400)
curl http://localhost:3000/api/users/not-a-uuid \
  -H "Authorization: Bearer <token>"

# Unknown route (404)
curl http://localhost:3000/api/nonexistent

# Wrong method (405)
curl -X PATCH http://localhost:3000/api/users \
  -H "Authorization: Bearer <token>"
```

---

## 📚 Key Node.js Concepts Demonstrated

1. **Centralized Error Handling** — `createError()` + `handleError()` pattern
2. **Information Leakage Prevention** — `expose` flag, generic 500 messages
3. **Request Logging** — `process.hrtime.bigint()`, `res.on('finish')` timing
4. **Input Validation** — UUID format, Content-Type regex, duplicate checks
5. **Error Propagation** — Throw from any layer, catch at the top
6. **Buffer-Based Body Parsing** — Memory-efficient `Buffer.concat`
7. **API Namespacing** — `/api/` prefix for clear route organization
8. **Defense in Depth** — Multiple layers of validation and error checking
