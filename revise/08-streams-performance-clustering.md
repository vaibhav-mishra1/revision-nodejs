# Question 08 — Streams, Performance, and Clustering

## 📋 What Was Asked

Create a Node.js application that: (1) processes a large file efficiently using streams, (2) demonstrates clustering to distribute requests across multiple worker processes, and (3) explains blocking vs non-blocking operations.

---

## 🏗️ Architecture & File Structure

```
question-08-streams-clustering/
├── stream-processing.js     ← Stream-based file processing
├── cluster-server.js        ← Primary process — forks workers
├── cluster-worker.js        ← HTTP server per worker
└── data/
    ├── large-input.txt      ← Source file (large text)
    └── processed-output.txt ← Generated output
```

Two separate programs: one for streams, one for clustering.

---

## 🔑 Part 1: Streams — Core Concepts

### What Are Streams?

Streams are objects that let you read/write data **incrementally** (chunk by chunk) instead of loading everything into memory at once.

```
Without Streams:              With Streams:
┌─────────────────┐           ┌──────────────┐
│  Read entire     │           │  Read chunk   │─→ Process → Write chunk
│  file into RAM   │           │  Read chunk   │─→ Process → Write chunk
│  (500MB = 500MB  │           │  Read chunk   │─→ Process → Write chunk
│   RAM used)      │           │  ...          │   (64KB RAM used)
│                  │           └──────────────┘
│  Process all     │
│  Write all       │
└─────────────────┘
```

### Four Types of Streams

| Type | Purpose | Example |
|------|---------|---------|
| **Readable** | Source of data | `fs.createReadStream()`, `req` |
| **Writable** | Destination for data | `fs.createWriteStream()`, `res` |
| **Transform** | Modify data as it passes through | `zlib.createGzip()`, custom transforms |
| **Duplex** | Both readable and writable | `net.Socket`, `crypto.createCipher` |

### Your Stream Implementation

```js
const { Transform } = require("stream");
const { pipeline } = require("stream/promises");

// Custom Transform stream — converts text to uppercase
const uppercaseTransform = new Transform({
    transform(chunk, encoding, callback) {
        callback(null, chunk.toString().toUpperCase());
        //       ↑ error    ↑ transformed data
    }
});

async function processLargeFile() {
    const readableStream = fs.createReadStream(inputFile, { encoding: "utf8" });
    const writableStream = fs.createWriteStream(outputFile);

    await pipeline(
        readableStream,      // Source: read from file
        uppercaseTransform,  // Transform: uppercase each chunk
        writableStream       // Sink: write to output file
    );
}
```

### Why `pipeline()` Instead of `.pipe()`?

| Feature | `.pipe()` | `pipeline()` |
|---------|-----------|--------------|
| Error handling | Must handle errors on EACH stream separately | Single try/catch catches all errors |
| Cleanup | Doesn't clean up streams on error (memory leak) | Automatically destroys all streams on error |
| Promise support | No (callback only) | `stream/promises` version returns a Promise |
| Backpressure | Handled | Handled |

```js
// ❌ Old way — error handling nightmare
readStream
    .pipe(transform)
    .pipe(writeStream);
readStream.on('error', handle);
transform.on('error', handle);
writeStream.on('error', handle);

// ✅ Modern way — clean and safe
await pipeline(readStream, transform, writeStream);
```

### The Transform Stream Callback

```js
transform(chunk, encoding, callback) {
    callback(null, chunk.toString().toUpperCase());
}
```

- **`chunk`** — A Buffer (or string if encoding is set) of data.
- **`encoding`** — The encoding of the chunk.
- **`callback(error, data)`** — Error-first callback. Pass `null` for no error, transformed data as second argument.

**Cross-Questions:**

- **Q: How big is each chunk?**
  - Default: 64KB for `createReadStream`. You can configure it with `highWaterMark`: `fs.createReadStream(file, { highWaterMark: 16 * 1024 })` for 16KB chunks.

- **Q: What's backpressure?**
  - When the writable stream can't keep up with the readable stream (writing to disk is slower than reading). `pipeline` handles this automatically — it pauses the readable stream until the writable is ready.

- **Q: Can you chain multiple transforms?**
  - Yes! `pipeline(read, transform1, transform2, transform3, write)`.

---

## 🔑 Part 2: Blocking vs Non-Blocking

### Blocking (Synchronous)
```js
const data = fs.readFileSync('file.txt');  // Thread is BLOCKED
console.log(data);                         // Runs only after file is fully read
// No other code can run during the read
```

### Non-Blocking (Asynchronous)
```js
fs.readFile('file.txt', (err, data) => {   // Registers callback, returns immediately
    console.log(data);                      // Runs when I/O completes
});
console.log('This runs FIRST');             // Event loop is free
```

**Why this matters for servers:**
```
Blocking server (1000 requests, 100ms per file read):
→ Total time: 100 seconds (sequential)

Non-blocking server (1000 requests, 100ms per file read):
→ Total time: ~100ms (concurrent — event loop handles all requests)
```

---

## 🔑 Part 3: Clustering — Core Concepts

### Why Clustering?

Node.js runs on a **single thread**. On a server with 8 CPU cores, a single Node process uses only 1 core (12.5% of available CPU).

Clustering forks multiple worker processes, each running on its own core:

```
┌─────────────────────────────────────────┐
│            Primary Process               │
│  (Manages workers, distributes connections│
│   via round-robin or OS scheduling)      │
├─────────┬─────────┬─────────┬───────────┤
│ Worker 1│ Worker 2│ Worker 3│ Worker 4  │
│ (Core 1)│ (Core 2)│ (Core 3)│ (Core 4) │
│ PID:1001│ PID:1002│ PID:1003│ PID:1004 │
│  :3000  │  :3000  │  :3000  │  :3000   │
└─────────┴─────────┴─────────┴───────────┘
         All share the same port!
```

### Your Cluster Implementation

```js
const cluster = require('cluster');
const os = require('os');

const WORKER_COUNT = Math.min(4, os.cpus().length);

if (cluster.isPrimary) {
    // This code runs in the PRIMARY process only
    for (let i = 0; i < WORKER_COUNT; i++) {
        cluster.fork();  // Create a worker
    }

    cluster.on('exit', (worker, code, signal) => {
        // Worker crashed → restart it
        if (code !== 0 || signal) {
            setTimeout(() => {
                cluster.fork();  // Note: should be cluster.fork() not cluster.fork
            }, 1000);
        }
    });
} else {
    // This code runs in WORKER processes
    require('./cluster-worker');
}
```

**How it works:**
1. `node cluster-server.js` starts the primary process.
2. `cluster.fork()` creates child processes that re-execute the SAME file.
3. In child processes, `cluster.isPrimary` is `false`, so they run the `else` branch.
4. Each worker starts an HTTP server on port 3000.
5. The OS (or Node's built-in round-robin) distributes connections across workers.

**Cross-Questions:**

- **Q: How can multiple processes listen on the same port?**
  - The primary process creates a TCP server socket and passes the file descriptor to workers. On Linux, `SO_REUSEPORT` allows this. Node's cluster module handles this automatically.

- **Q: Why `Math.min(4, os.cpus().length)`?**
  - Don't create more workers than CPU cores. Extra workers cause context switching overhead. We also cap at 4 for dev environments where it's unnecessary to use all cores.

- **Q: What happens when a worker crashes?**
  - The `cluster.on('exit')` event fires. We check if the exit was abnormal (`code !== 0` or killed by a signal) and restart the worker. Normal exits (code 0) are intentional shutdowns.

- **Q: Why delay 1 second before restarting?**
  - Prevents a crash loop. If a worker crashes immediately after starting (e.g., port conflict), without a delay you'd get infinite rapid restarts consuming all CPU.

- **Q: Do workers share memory?**
  - No! Each worker is a separate OS process with its own memory space. They don't share variables or state. For shared state, use Redis, a database, or IPC (Inter-Process Communication).

### Worker HTTP Server

```js
const server = http.createServer((req, res) => {
    res.end(JSON.stringify({
        message: "Hello from a cluster worker!",
        processId: process.pid    // Shows which worker handled the request
    }));
});
```

Including `process.pid` proves that different requests are handled by different workers.

---

## 🔄 Clustering vs Other Scaling Approaches

| Approach | How | When to Use |
|----------|-----|-------------|
| **Clustering** (`cluster` module) | Fork multiple Node processes on one machine | CPU-bound workloads on multi-core machines |
| **PM2** | Process manager with built-in clustering | Production deployment, zero-downtime restarts |
| **Worker Threads** | Threads within a single process (shared memory) | CPU-intensive tasks (image processing, crypto) |
| **Horizontal Scaling** | Multiple machines behind a load balancer | When one machine isn't enough |
| **Child Processes** | `child_process.spawn/fork` | Running other programs/scripts |

---

## 🎯 Cross-Questions to Prepare For

| Question | Your Answer |
|----------|------------|
| When would you use streams over `readFile`? | When files are large (> 100MB), or when you need to start processing before the entire file is read, or when you're proxying data between sources. |
| What's the default chunk size? | 64KB (`highWaterMark: 65536` bytes) for file read streams. 16KB for other streams. |
| What's backpressure? | When the consumer can't process data as fast as the producer sends it. `pipeline()` handles this by pausing the readable stream. |
| Why not just use threads? | Node's `cluster` module is simpler for HTTP servers. Worker threads share memory (complex synchronization). Clusters are separate processes — safer, simpler. |
| What's the difference between `cluster.fork()` and `child_process.fork()`? | `cluster.fork()` creates a worker that can share a server port. `child_process.fork()` creates a generic child process for running scripts. |
| How does PM2 relate to clustering? | PM2 uses `cluster` module internally. `pm2 start app.js -i max` starts one worker per CPU core, handles restarts, and provides monitoring. |
| What if a worker runs out of memory? | The OS kills it (OOM killer). The primary's `exit` handler detects this and restarts it. Set `--max-old-space-size` to control Node's heap limit. |

---

## 🔧 How to Run

```bash
cd question-08-streams-clustering

# Stream processing
node stream-processing.js

# Clustering
node cluster-server.js
```

## 🧪 Test Clustering

```bash
# Send multiple requests — notice different PIDs in responses
curl http://localhost:3000/
curl http://localhost:3000/
curl http://localhost:3000/
curl http://localhost:3000/
```

---

## 📚 Key Node.js Concepts Demonstrated

1. **Readable/Writable/Transform Streams** — Incremental data processing
2. **`pipeline()`** — Safe stream chaining with automatic cleanup
3. **Memory Efficiency** — Processing GB-sized files with KB-sized memory
4. **`cluster` Module** — Multi-process architecture
5. **Worker Management** — Automatic restart on crash
6. **Blocking vs Non-Blocking** — Sync vs async I/O impact on throughput
7. **`os.cpus()`** — Detecting available CPU cores
