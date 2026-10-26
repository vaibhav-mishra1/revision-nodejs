# Question 02 — Asynchronous Programming and the Event Loop

## 📋 What Was Asked

Create a program demonstrating synchronous vs. asynchronous execution. Show callbacks, Promises, and async/await. Use timers and I/O to demonstrate event loop execution order.

---

## 🏗️ File Structure

```
question-02-async/
├── index.js      ← All demonstrations in one file
└── sample.txt    ← Test file for I/O operations
```

---

## 🔑 Core Concepts You MUST Know

### 1. The Node.js Event Loop — The Single Most Important Concept

Node.js is **single-threaded** but **non-blocking**. The event loop processes tasks in a specific order:

```
┌──────────────────────────────────────────┐
│               Call Stack                  │
│  (Synchronous code runs here first)       │
└──────────────┬───────────────────────────┘
               ↓
┌──────────────────────────────────────────┐
│          Microtask Queue                  │
│  (process.nextTick → Promise.then)        │
│  Drained COMPLETELY before next phase     │
└──────────────┬───────────────────────────┘
               ↓
┌──────────────────────────────────────────┐
│           Timers Phase                    │
│  (setTimeout, setInterval callbacks)      │
└──────────────┬───────────────────────────┘
               ↓
┌──────────────────────────────────────────┐
│       Pending Callbacks Phase             │
│  (System-level callbacks, TCP errors)     │
└──────────────┬───────────────────────────┘
               ↓
┌──────────────────────────────────────────┐
│         Poll Phase                        │
│  (I/O callbacks — fs.readFile, etc.)      │
└──────────────┬───────────────────────────┘
               ↓
┌──────────────────────────────────────────┐
│          Check Phase                      │
│  (setImmediate callbacks)                 │
└──────────────┬───────────────────────────┘
               ↓
┌──────────────────────────────────────────┐
│      Close Callbacks Phase                │
│  (socket.on('close'), etc.)               │
└──────────────────────────────────────────┘
```

### 2. Execution Priority (Memorize This!)

```
1. Synchronous code          ← Runs first, always
2. process.nextTick()        ← Microtask, runs before all async
3. Promise.resolve().then()  ← Microtask, runs after nextTick
4. setTimeout(fn, 0)         ← Timer phase (macro-task)
5. setImmediate()            ← Check phase (macro-task)
6. I/O callbacks             ← Poll phase
```

> **Critical Rule:** Microtasks (nextTick, Promises) are drained COMPLETELY between every macro-task phase. One microtask can schedule another, and it'll run before the next phase.

---

## 🧪 What Each Demo Function Shows

### Demo 1: `readSynchronously()` — Blocking Execution

```js
console.log("Before sync read");
const data = fs.readFileSync(filePath, 'utf-8');   // ← BLOCKS HERE
console.log("Sync result", data.trim());
console.log("After sync read");
```

**Output order:** `Before → Sync result → After` (sequential, predictable)

**Why it blocks:** `readFileSync` tells Node: "Don't do anything else until this file is read." The event loop is frozen.

**Cross-Question:** *When would you use sync operations?*
- At application startup (reading config files).
- In CLI tools where sequential execution is fine.
- NEVER in a server handling concurrent requests.

### Demo 2: `readWithCallback()` — Classic Async

```js
console.log("Before callback read");
fs.readFile(filePath, 'utf-8', (error, data) => {
    console.log("Callback result", data.trim());  // ← Runs later (I/O poll phase)
});
console.log("After callback read");               // ← Runs immediately
```

**Output order:** `Before → After → Callback result`

**Why?** `fs.readFile` is non-blocking. It registers the callback and returns immediately. "After" runs first because the callback only executes when the I/O operation completes and the event loop reaches the poll phase.

### Demo 3: `readWithPromise()` — Promise Chain

```js
console.log("Before promise read");
return fsPromises.readFile(filePath, 'utf-8')
    .then((data) => {
        console.log("Promise result", data.trim());
    })
    .catch((error) => {
        console.error("Promise read failed:", error.message);
    });
```

**Key insight:** `.then()` callbacks go into the **microtask queue**, which has higher priority than timers or I/O callbacks.

### Demo 4: `readWithAsyncAwait()` — Modern Syntax

```js
const data = await fsPromises.readFile(filePath, 'utf-8');
console.log("Async/await result", data.trim());
console.log("After async/await read");
```

**Output order:** `Before → Async/await result → After` (appears synchronous!)

**Why `await` looks blocking but isn't:**
- `await` pauses only the **current async function**.
- The event loop continues processing other tasks.
- When the Promise resolves, the function resumes via the microtask queue.

### Demo 5: `demonstrateEventLoop()` — The Money Demo 🎯

```js
console.log("1. Before scheduling I/O");

fs.readFile(filePath, "utf8", (error) => {
    console.log("3. I/O callback");

    setTimeout(() => {
        console.log("6. setTimeout callback");
    }, 0);

    setImmediate(() => {
        console.log("5. setImmediate callback");
    });

    Promise.resolve().then(() => {
        console.log("4. Promise microtask");
    });
});

console.log("2. After scheduling I/O");
```

**Expected Output:**
```
1. Before scheduling I/O
2. After scheduling I/O        ← Sync code finishes first
3. I/O callback                ← I/O completes, callback fires
4. Promise microtask           ← Microtask drains before next phase
5. setImmediate callback       ← Check phase (inside I/O callback, setImmediate fires before setTimeout)
6. setTimeout callback         ← Timer phase
```

**Why this order?**

| Step | Why |
|------|-----|
| 1 & 2 | Synchronous code runs first |
| 3 | I/O callback fires when file read completes |
| 4 | Inside the I/O callback, `Promise.resolve().then()` goes to the microtask queue. Microtasks drain BEFORE the event loop moves to the next phase |
| 5 | `setImmediate` fires in the **check phase**. When inside an I/O callback, `setImmediate` ALWAYS fires before `setTimeout(fn, 0)` |
| 6 | `setTimeout(fn, 0)` fires in the next **timers phase** |

---

## ⚡ The `setImmediate` vs `setTimeout(fn, 0)` Rule

```
Inside an I/O callback:
  setImmediate → fires FIRST (check phase comes right after poll)
  setTimeout   → fires SECOND (timers phase is in the next loop iteration)

Outside an I/O callback (top-level):
  Order is NON-DETERMINISTIC (depends on system timing)
```

This is one of the most commonly asked questions about the event loop.

---

## 🎯 Cross-Questions to Prepare For

| Question | Your Answer |
|----------|------------|
| What is the event loop? | It's Node's mechanism for handling async operations. It runs in phases, each processing a specific type of callback. It allows Node to be non-blocking despite being single-threaded. |
| Why is Node single-threaded? | The main JavaScript execution thread is single-threaded. However, Node uses a thread pool (via libuv) for I/O operations, DNS lookups, and crypto operations. This gives the benefits of concurrency without the complexity of thread synchronization. |
| What are microtasks? | `process.nextTick()` and `Promise` callbacks. They drain completely between every event loop phase. They have higher priority than any macro-task (timers, I/O). |
| What's the difference between `process.nextTick()` and `Promise.then()`? | Both are microtasks, but `nextTick` has higher priority. `nextTick` callbacks run before Promise callbacks. Overusing `nextTick` can starve the event loop. |
| Can `async/await` cause blocking? | The `await` keyword pauses only the async function, not the event loop. However, if you `await` a CPU-intensive synchronous operation wrapped in a Promise, the event loop IS blocked during that computation. |
| What's callback hell and how do Promises solve it? | Nested callbacks become unreadable. Promises allow chaining (`.then().then()`) and `async/await` makes async code look synchronous. |
| How do you handle errors in each pattern? | Callbacks: error-first pattern `(err, data)`. Promises: `.catch()`. Async/await: `try/catch`. |
| What's the thread pool? | libuv maintains 4 threads (default) for file I/O, DNS, compression. You can change it with `UV_THREADPOOL_SIZE`. |

---

## 🔧 How to Run

```bash
cd question-02-async
node index.js
```

---

## 📚 Key Node.js Concepts Demonstrated

1. **Event Loop Phases** — Timers → Poll → Check → Microtasks between each
2. **Sync vs Async I/O** — `readFileSync` vs `readFile` vs `fs/promises`
3. **Three Async Patterns** — Callbacks, Promises (`.then/.catch`), async/await
4. **Microtask Priority** — `Promise.then()` runs before `setTimeout(0)` and `setImmediate`
5. **setImmediate vs setTimeout(0)** — Deterministic inside I/O callbacks, non-deterministic outside
6. **Error Handling** — Error-first callbacks, `.catch()`, `try/catch`
