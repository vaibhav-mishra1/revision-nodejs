# Question 01 — Node.js Modules and File Operations

## 📋 What Was Asked

Create a modular Node.js application that manages a collection of records stored in a JSON file using CommonJS modules. Implement CRUD (Create, Read, Update, Delete) operations with proper error handling and data persistence.

---

## 🏗️ Architecture & File Structure

```
question-01-modules/
├── app.js            ← Entry point — orchestrates CRUD demo
├── fileStorage.js    ← Low-level file I/O (read/write JSON)
├── recordService.js  ← Business logic (CRUD functions)
├── test-storage.js   ← Quick smoke-test for the storage layer
└── data.json         ← Persistent JSON data store
```

### Why This Structure?

| Layer | File | Responsibility |
|-------|------|---------------|
| **Storage** | `fileStorage.js` | Only knows about reading/writing files. Has zero business logic. |
| **Service** | `recordService.js` | All business rules — validation, ID generation, timestamps. Calls storage layer. |
| **App** | `app.js` | Just a runner — calls service functions and prints results. |

> **Key Design Decision:** Separation of Concerns. The storage layer doesn't know what a "record" looks like, and the service layer doesn't know the file path. This means you could swap the JSON file for a database later by changing only `fileStorage.js`.

---

## 🔑 Core Concepts You Must Know

### 1. CommonJS Modules (`require` / `module.exports`)

```js
// Exporting
module.exports = { readData, writeData };

// Importing
const { readData, writeData } = require('./fileStorage');
```

**Cross-Questions to Expect:**
- **Q: Why CommonJS and not ES Modules (`import/export`)?**
  - The assignment explicitly asked for CommonJS. Node.js historically used CommonJS. ES Modules are supported (since Node 12+), but you need `"type": "module"` in `package.json` or use `.mjs` extension. We set `"type": "commonjs"` in our `package.json`.

- **Q: What's the difference between `module.exports` and `exports`?**
  - `exports` is just a reference to `module.exports`. If you assign `exports = something`, it breaks the reference. Always use `module.exports` when exporting an entire object.

- **Q: How does `require()` caching work?**
  - Node caches the result of `require()` after the first call. Subsequent `require()` calls for the same module return the cached version. That's why in-memory data arrays shared across modules stay consistent.

### 2. `fs/promises` — The Async File System API

```js
const fs = require('fs/promises');
```

**Why `fs/promises` instead of `fs` with callbacks?**
- Cleaner code — no callback hell.
- Works natively with `async/await`.
- The `fs` callback API (`fs.readFile(path, cb)`) is older and harder to compose.

**Cross-Questions:**
- **Q: Why not `fs.readFileSync`?**
  - Sync operations block the event loop. In a server context, this would freeze all other requests while reading a file. Even in a CLI tool, async is a better habit.

- **Q: What's `fs/promises` vs `require('fs').promises`?**
  - Same thing. `fs/promises` is a cleaner direct import available since Node 14.

### 3. Error Handling Strategy

```js
// In fileStorage.js
if (error.code === 'ENOENT') {
    return [];  // File doesn't exist yet → return empty array
}
if (error instanceof SyntaxError) {
    throw new Error('Invalid JSON format in data.json');
}
```

| Error | How We Handle It | Why |
|-------|-----------------|-----|
| `ENOENT` (file not found) | Return empty array `[]` | App should work even on first run |
| `SyntaxError` (bad JSON) | Throw descriptive error | Don't silently ignore corrupted data |
| Write validation | Check `Array.isArray()` before writing | Prevent corrupt data from being written |

### 4. UUID Generation

```js
const { randomUUID } = require('crypto');
const id = randomUUID(); // "550e8400-e29b-41d4-a716-446655440000"
```

**Why `crypto.randomUUID()` and not `Date.now()` or auto-increment?**
- UUIDs are globally unique — no collisions even across distributed systems.
- Auto-increment IDs leak information (how many records exist, creation order).
- `crypto.randomUUID()` is built into Node (since v19), no external dependency needed.

### 5. Data Persistence

Every mutation (create/update/delete) follows this pattern:
```
Read all records → Modify in memory → Write entire array back to file
```

**Cross-Question:**
- **Q: Isn't reading the entire file every time inefficient?**
  - For a JSON file store, yes. This is fine for small datasets. For production, you'd use a database. The assignment specifically asks for JSON file storage to test your file I/O understanding.

- **Q: What happens if two requests hit at the same time (race condition)?**
  - In this single-process CLI app, it's sequential. In a server context, you could get race conditions. Solutions include file locking (`fs-ext`) or switching to a database with ACID guarantees.

---

## 💡 Logic Walkthrough

### Create Record
1. Validate input (must be an object, must have non-empty `name`).
2. Read all existing records from file.
3. Generate UUID and timestamp.
4. Spread user fields, overriding `name` (trimmed) and adding `id` + `createdAt`.
5. Push to array and write back.

### Update Record
1. Validate `id` and `updatedFields`.
2. Find record by index (not just `.find()` — need the index for in-place update).
3. Spread existing record, then spread updates (new fields override old ones).
4. Add `updatedAt` timestamp.
5. Replace at index and write back.

### Delete Record
1. Find by index.
2. Use `splice(index, 1)` — removes in place and returns the removed element.
3. Write the remaining array back.

---

## 🎯 Cross-Questions to Prepare For

| Question | Your Answer |
|----------|------------|
| Why separate `fileStorage` and `recordService`? | Single Responsibility Principle. Storage handles I/O, service handles business logic. Swapping storage backends requires changing only one file. |
| Why `async/await` instead of callbacks? | Readability, easier error handling with `try/catch`, avoids callback hell. |
| What if `data.json` doesn't exist? | `readData()` catches `ENOENT` and returns `[]`. First write creates the file automatically. |
| What if `data.json` has invalid JSON? | We catch `SyntaxError` and throw a meaningful error instead of crashing. |
| Why `Array.isArray()` check in `writeData`? | Prevents accidentally overwriting the file with non-array data, which would corrupt the store. |
| Why spread operator for updates? | `{ ...existing, ...updates }` preserves unchanged fields while overriding changed ones. Immutable update pattern. |
| Why trim the name? | Prevents storing `"  John  "` as a name. Defensive input sanitization. |
| Could you add pagination? | Yes — accept `page` and `limit` params, use `Array.slice(offset, offset + limit)`. |

---

## 🔧 How to Run

```bash
cd question-01-modules
node app.js
```

## ✅ Expected Output

```
New record created: { id: '...', name: 'Sample Record 3', ... }
All records: [...]
Record by ID: { id: '...', name: 'Sample Record 3', ... }
Updated: { id: '...', description: 'Updated description', updatedAt: '...' }
Deleted record: { id: '...', ... }
Records after deletion: [...]
```

---

## 📚 Key Node.js Concepts Demonstrated

1. **CommonJS Module System** — `require()`, `module.exports`, module caching
2. **`fs/promises`** — Async file operations without callback hell
3. **`path.join(__dirname, ...)`** — Platform-safe path resolution
4. **`crypto.randomUUID()`** — Built-in UUID generation
5. **Separation of Concerns** — Storage vs. business logic
6. **Defensive Programming** — Type checks, empty string checks, array validation
7. **Error propagation** — Catch at the right level, re-throw with context
