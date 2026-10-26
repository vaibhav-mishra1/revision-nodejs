# Question 09 — Nginx Configuration and Reverse Proxy

## 📋 What Was Asked

Configure Nginx to serve static files and forward API requests to a Node.js server. Demonstrate understanding of reverse proxying, header forwarding, and the separation of static vs dynamic content.

---

## 🏗️ Architecture & File Structure

```
question-09-nginx/
├── server.js                  ← Node.js API server (port 3000)
├── public/
│   ├── index.html             ← Static HTML page
│   └── style.css              ← Static CSS
├── nginx/
│   └── question-09.conf       ← Nginx configuration file
└── README.md
```

### Architecture Diagram

```
                    Internet / Client (Browser)
                              │
                              ▼
                    ┌──────────────────┐
                    │     Nginx        │  Port 80
                    │  (Reverse Proxy  │
                    │   + Static Files)│
                    └──────┬───────────┘
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
    ┌──────────────────┐    ┌──────────────────┐
    │  Static Files    │    │  Node.js Server  │  Port 3000
    │  /var/www/q09/   │    │  (API only)      │
    │  index.html      │    │  /api/hello      │
    │  style.css       │    │  /api/*          │
    └──────────────────┘    └──────────────────┘
```

---

## 🔑 Core Concepts You MUST Know

### 1. What Is a Reverse Proxy?

A **reverse proxy** sits between clients and your backend servers. The client talks to the proxy, and the proxy forwards the request to the appropriate backend.

**Forward Proxy vs Reverse Proxy:**

| Aspect | Forward Proxy | Reverse Proxy |
|--------|--------------|---------------|
| **Who uses it** | Client-side (e.g., corporate firewall, VPN) | Server-side (e.g., Nginx, HAProxy) |
| **Purpose** | Client wants to hide its identity or bypass restrictions | Server wants to distribute load, cache, or handle SSL |
| **Client knows about it** | Yes — client configures the proxy | No — client thinks it's talking to the real server |

**Why use a reverse proxy in front of Node.js?**

| Benefit | Explanation |
|---------|-------------|
| **Static file serving** | Nginx serves files 10-100x faster than Node.js for static content |
| **SSL termination** | Nginx handles HTTPS certificates; Node.js gets plain HTTP |
| **Load balancing** | Nginx can distribute requests across multiple Node.js instances |
| **Security** | Node.js isn't directly exposed to the internet |
| **Caching** | Nginx can cache API responses to reduce Node.js load |
| **Rate limiting** | Nginx can block abusive traffic before it reaches Node.js |
| **Compression** | Nginx handles gzip/brotli compression more efficiently |

### 2. The Nginx Configuration (What Should Be in `question-09.conf`)

```nginx
server {
    listen 80;
    server_name localhost;

    # Serve static files from /var/www/question-09
    root /var/www/question-09;
    index index.html;

    # Static file requests — try file, then directory, then 404
    location / {
        try_files $uri $uri/ =404;
    }

    # API requests — forward to Node.js
    location /api/ {
        proxy_pass http://127.0.0.1:3000;

        # Forward important headers
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 3. Configuration Directives Explained

| Directive | What It Does |
|-----------|-------------|
| `listen 80` | Nginx listens on port 80 (standard HTTP) |
| `server_name localhost` | Which domain this config applies to |
| `root /var/www/question-09` | Directory where static files live |
| `index index.html` | Default file when requesting a directory |
| `try_files $uri $uri/ =404` | Try exact file → try directory → return 404 |
| `location /api/` | Match all URLs starting with `/api/` |
| `proxy_pass` | Forward matched requests to Node.js |

### 4. Header Forwarding — Why It Matters

When Nginx forwards a request to Node.js, the original client information is lost (Node.js sees the request as coming from Nginx). We forward these headers to preserve it:

```nginx
proxy_set_header Host              $host;
proxy_set_header X-Real-IP         $remote_addr;
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

| Header | Purpose |
|--------|---------|
| `Host` | The original domain the client requested. Without this, Node sees `127.0.0.1:3000` |
| `X-Real-IP` | The client's actual IP address |
| `X-Forwarded-For` | Chain of proxies the request passed through |
| `X-Forwarded-Proto` | `http` or `https` — tells Node if the original request was encrypted |

**Cross-Question:** *Why does Node.js need the client's real IP?*
- Rate limiting (per-IP), logging, geolocation, security auditing. Without these headers, all requests appear to come from `127.0.0.1`.

### 5. The Node.js API Server

```js
const http = require("http");

const HOST = "127.0.0.1";   // Only accepts local connections
const PORT = 3000;

const server = http.createServer((req, res) => {
    if (req.url === "/api/hello" && req.method === "GET") {
        res.writeHead(200, { "Content-Type": "application/json" });
        return res.end(JSON.stringify({
            success: true,
            message: "Hello from Node.js API"
        }));
    }

    res.writeHead(404, { "Content-Type": "application/json" });
    res.end(JSON.stringify({
        success: false,
        message: "API route not found"
    }));
});
```

**Why `HOST = "127.0.0.1"` and not `"0.0.0.0"`?**
- `127.0.0.1` = only accepts connections from localhost (the same machine).
- `0.0.0.0` = accepts connections from anywhere.
- Since Nginx is on the same machine, Node.js doesn't need to be accessible from the outside. This adds a security layer — even if someone bypasses Nginx, they can't reach Node.js directly from the network.

### 6. `try_files` Directive

```nginx
location / {
    try_files $uri $uri/ =404;
}
```

This is how Nginx decides what to serve:
1. `$uri` — Is there a file matching the URL? (`/style.css` → serve the file)
2. `$uri/` — Is there a directory? (check for `index.html` in it)
3. `=404` — Nothing found → return 404

### 7. Static Files vs Dynamic Content

| Static | Dynamic |
|--------|---------|
| HTML, CSS, JS, images, fonts | API responses, database queries |
| Same content for every request | Content varies per request/user |
| Nginx serves directly from disk | Node.js generates on the fly |
| Can be cached aggressively | Caching depends on the content |
| **Very fast** (just file read) | **Slower** (computation + I/O) |

---

## 📋 Nginx Setup Commands (For Reference)

```bash
# Install Nginx
sudo apt update && sudo apt install nginx

# Copy static files
sudo mkdir -p /var/www/question-09
sudo cp -r public/. /var/www/question-09/

# Create Nginx config
sudo cp nginx/question-09.conf /etc/nginx/sites-available/question-09

# Enable the site
sudo ln -s /etc/nginx/sites-available/question-09 /etc/nginx/sites-enabled/question-09

# Remove default site (optional)
sudo rm /etc/nginx/sites-enabled/default

# Test configuration
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx
```

---

## 🎯 Cross-Questions to Prepare For

| Question | Your Answer |
|----------|------------|
| What is a reverse proxy? | A server that sits between clients and backend servers. Forwards requests, hides backend details, can load balance, cache, and terminate SSL. |
| Why not serve static files from Node.js? | Nginx is written in C, event-driven, and optimized for file serving. It's 10-100x faster than Node.js for static files. Node.js's strength is dynamic content and I/O. |
| What's the difference between `proxy_pass` and `redirect`? | `proxy_pass` forwards the request internally (client doesn't know). A redirect (301/302) tells the client to make a new request to a different URL. |
| What's `try_files`? | Nginx tries each argument in order. `$uri` = exact file, `$uri/` = directory, `=404` = fallback. First match wins. |
| How would you add HTTPS? | Add SSL certificate to Nginx: `listen 443 ssl; ssl_certificate /path/to/cert; ssl_certificate_key /path/to/key;`. Redirect HTTP (port 80) to HTTPS. |
| How would you add load balancing? | Use Nginx `upstream` block: `upstream backend { server 127.0.0.1:3000; server 127.0.0.1:3001; }` then `proxy_pass http://backend;` |
| What's the difference between `sites-available` and `sites-enabled`? | `sites-available` stores all config files. `sites-enabled` has symlinks to active configs. Deactivate a site by removing its symlink, without deleting the config. |
| What happens if Node.js is down? | Nginx returns `502 Bad Gateway` — it can't connect to the backend. You can configure a fallback page with `error_page 502 /502.html;`. |
| Why forward `X-Forwarded-Proto`? | So Node.js knows if the original request was HTTPS. Useful for generating correct redirect URLs and enforcing HTTPS-only policies. |

---

## 🔧 How to Run

```bash
# 1. Start Node.js API server
cd question-09-nginx
node server.js

# 2. Configure and start Nginx (separate terminal)
sudo nginx -t
sudo systemctl start nginx
```

## 🧪 Test Cases

```bash
# Static file (served by Nginx)
curl http://localhost/
curl http://localhost/style.css

# API request (forwarded to Node.js by Nginx)
curl http://localhost/api/hello

# Non-existent API route (Node.js returns 404)
curl http://localhost/api/missing

# Non-existent static file (Nginx returns 404)
curl http://localhost/nonexistent.html
```

---

## 📚 Key Concepts Demonstrated

1. **Reverse Proxy** — Nginx forwards API requests to Node.js
2. **Static File Serving** — Nginx serves HTML/CSS/JS directly
3. **Header Forwarding** — Preserving client IP and protocol information
4. **Port Separation** — Nginx on port 80, Node.js on port 3000 (internal only)
5. **`try_files`** — Nginx's file resolution strategy
6. **Security** — Node.js bound to `127.0.0.1` (not internet-accessible)
7. **Separation of Concerns** — Nginx handles what it's best at, Node handles dynamic logic
