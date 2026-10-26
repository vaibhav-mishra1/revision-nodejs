# Question 05 — REST API with CRUD Operations

## 📋 What Was Asked

Build a complete REST API for managing users. Store data in a JSON file. Support full CRUD with unique IDs, validation, persistent storage, and proper HTTP methods/status codes.

---

## 🏗️ Architecture & File Structure

```
question-05-rest-api/
├── server.js                  ← Entry point + global error handler
├── routes/
│   └── userRoutes.js          ← URL matching + method dispatch
├── services/
│   └── userService.js         ← Business logic + file persistence + validation
├── utils/
│   └── httpUtils.js           ← sendJson() + readJsonBody()
└── data/
    └── users.json             ← Persistent JSON file store
```

---

## 🔑 Core Concepts You MUST Know

### 1. REST Principles (Representational State Transfer)

REST is not a protocol — it's an **architectural style**. Your API follows these conventions:

| Principle | How We Implement It |
|-----------|-------------------|
| **Resources are identified by URLs** | `/users` (collection), `/users/:id` (individual) |
| **HTTP methods define actions** | GET = read, POST = create, PUT = update, DELETE = delete |
| **Stateless** | Each request contains all info needed (no server-side session) |
| **Uniform interface** | Consistent response format, standard status codes |
| **Representations** | Data transferred as JSON (`Content-Type: application/json`) |

### 2. CRUD → HTTP Method Mapping

| CRUD Operation | HTTP Method | Endpoint | Status Code |
|---------------|-------------|----------|-------------|
| **Create** | `POST` | `/users` | `201 Created` |
| **Read All** | `GET` | `/users` | `200 OK` |
| **Read One** | `GET` | `/users/:id` | `200 OK` / `404` |
| **Update** | `PUT` | `/users/:id` | `200 OK` / `404` |
| **Delete** | `DELETE` | `/users/:id` | `200 OK` / `404` |

**Cross-Questions:**
- **Q: Why PUT and not PATCH for update?**
  - `PUT` replaces the entire resource (you must send ALL fields). `PATCH` partially updates (send only changed fields). Our implementation requires full `name` and `email`, so `PUT` is correct. If we allowed partial updates, `PATCH` would be more appropriate.

- **Q: Why POST to `/users` and not PUT?**
  - `POST` = "create a new resource, server assigns the ID." `PUT` = "create or replace at a specific URL." Since the client doesn't know the ID yet, `POST` is correct.

### 3. Dynamic Route Matching with Regex

```js
const match = pathname.match(/^\/users\/([^/]+)$/);
if (match) {
    const id = match[1];
    // Handle individual user operations
}
```

**Why regex instead of `pathname.split('/')`?**
- Regex is more precise — `/users/abc` matches, but `/users/abc/extra` doesn't.
- The capture group `([^/]+)` extracts the ID automatically.
- It's the pattern that frameworks like Express use internally for `:param`.

**The regex explained:**
```
^\/users\/   → Must start with "/users/"
([^/]+)      → Capture one or more non-slash characters (the ID)
$            → Must end here (no trailing segments)
```

### 4. JSON File Persistence — Read-Modify-Write Pattern

Every write operation follows:
```
1. Read entire file → JSON.parse
2. Modify the in-memory array
3. JSON.stringify → Write entire file
```

```js
async function createUser(data) {
    const users = await getAllUsers();    // 1. Read
    const user = { id: randomUUID(), ... };
    users.push(user);                    // 2. Modify
    await saveUsers(users);              // 3. Write
    return user;
}
```

**Cross-Questions:**

- **Q: This reads the entire file for every operation. Isn't that slow?**
  - Yes, for large datasets. JSON file storage is fine for learning/prototyping (< 1000 records). For production, use a database (MongoDB, PostgreSQL) that supports indexed lookups and atomic operations.

- **Q: What about concurrent write conflicts?**
  - If two POST requests arrive simultaneously, both read the same file, add their user, and write back. One user could be lost (last-write-wins). Solutions: file locking, queueing writes, or using a database.

- **Q: Why `JSON.stringify(users, null, 2)`?**
  - `null` = no replacer function (include all fields). `2` = indent with 2 spaces for human readability. Without indentation, the file would be one long line.

### 5. Validation at the Service Layer

```js
function validateUser(data) {
    if (data === null || typeof data !== 'object' || Array.isArray(data)) {
        return 'User data must be a valid JSON object';
    }
    if (typeof data.name !== 'string' || !data.name.trim()) {
        return 'Name is required and must be a non-empty string';
    }
    if (typeof data.email !== 'string' ||
        !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(data.email)) {
        return 'A valid email address is required';
    }
    return null;
}
```

**Why validate in the service layer?**
- The service is the "gatekeeper" before data touches the storage.
- If you add another interface (CLI, GraphQL), validation still applies.
- The route handler stays thin — it just calls the service.

**Custom Error Pattern:**
```js
const err = new Error(error);
err.statusCode = 400;
throw err;
```

We attach `statusCode` to the Error object so the global error handler knows what HTTP status to return. This is a common Node.js pattern — it's how Express's `createError` from `http-errors` works.

### 6. Global Error Handler

```js
const server = http.createServer(async (req, res) => {
    try {
        await handleRoutes(req, res);
    } catch (error) {
        return sendJson(res, error.statusCode || 500, {
            success: false,
            message: error.statusCode ? error.message : 'Internal server error'
        });
    }
});
```

**Key design decision:**
- If `error.statusCode` exists → it's a **known, intentional error** (validation failed, not found). Show its message.
- If no `statusCode` → it's an **unexpected bug**. Return generic "Internal server error." Never expose stack traces or internal details to clients.

---

## 💡 Key Implementation Decisions

### Why `randomUUID()` for IDs?
- Globally unique, no collisions.
- No sequential guessing (security benefit).
- Built into Node.js `crypto` module — no dependencies.

### Why `return sendJson(...)` in Routes?
- The `return` stops the route handler from falling through to the next `if` block.
- Without it, after matching `/users`, the code would continue to the 404 block.

### Why `findIndex` Instead of `find` for Update/Delete?
- `find` returns the object but not its position. We need the index to:
  - Replace in-place: `records[index] = updatedRecord`
  - Remove: `records.splice(index, 1)`

---

## 🎯 Cross-Questions to Prepare For

| Question | Your Answer |
|----------|------------|
| What is REST? | An architectural style for APIs. Resources identified by URLs, actions through HTTP methods, stateless, uniform interface, data as representations (JSON). |
| Difference between REST and RESTful? | REST is the set of principles. "RESTful" means an API that follows those principles. |
| Why JSON for data storage instead of a DB? | Assignment requirement. Demonstrates file I/O skills. For production, use a database for indexing, transactions, concurrent access, and scalability. |
| How would you add pagination? | Accept `?page=1&limit=10` query params. Slice the array: `users.slice((page-1) * limit, page * limit)`. Return total count and page info in response. |
| How would you add search/filter? | Accept `?name=Alice` query param. Filter: `users.filter(u => u.name.includes(query))`. |
| What's idempotent? | An operation that produces the same result when called multiple times. GET, PUT, DELETE are idempotent. POST is not (each call creates a new resource). |
| What if two users have the same email? | Currently allowed. Should add unique email check: `users.some(u => u.email === data.email)`. (Fixed in Q10.) |
| Why not use Express? | Assignment requirement. Shows understanding of what Express abstracts away — body parsing, routing, error handling, middleware pipeline. |

---

## 🔧 How to Run

```bash
cd question-05-rest-api
node server.js
```

## 🧪 Test with curl

```bash
# List all users
curl http://localhost:3000/users

# Create a user
curl -X POST http://localhost:3000/users \
  -H "Content-Type: application/json" \
  -d '{"name": "Alice", "email": "alice@test.com"}'

# Get user by ID (replace with actual UUID)
curl http://localhost:3000/users/<uuid>

# Update user
curl -X PUT http://localhost:3000/users/<uuid> \
  -H "Content-Type: application/json" \
  -d '{"name": "Alice Updated", "email": "alice@test.com"}'

# Delete user
curl -X DELETE http://localhost:3000/users/<uuid>
```

---

## 📚 Key Node.js Concepts Demonstrated

1. **REST Architecture** — Proper resource naming, HTTP method semantics
2. **Dynamic Routing** — Regex-based URL parameter extraction
3. **JSON File Persistence** — Read-modify-write pattern
4. **Async/Await Server** — `async` request handler with `try/catch`
5. **Custom Error Objects** — Attaching `statusCode` for centralized handling
6. **Input Validation** — Type checks, format validation, defensive programming
7. **UUID Generation** — `crypto.randomUUID()` for unique identifiers
