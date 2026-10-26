# Question 07 — Session-Based Authentication vs. JWT

## 📋 What Was Asked

Demonstrate the differences between session-based and JWT-based authentication. Implement a session-based login flow. Compare both approaches — covering identity verification, storage, expiration, logout, and security.

---

## 🏗️ Architecture & File Structure

```
question-07-session-based-auth/
├── server.js                    ← Entry point, URL parsing, error handling
├── routes/
│   ├── authRoutes.js            ← POST /login, POST /logout
│   └── userRoutes.js            ← GET /profile (protected)
├── services/
│   └── sessionService.js        ← Session store (in-memory Map)
└── utils/
    └── httpUtils.js             ← sendJson + readJsonBody
```

---

## 🔑 Core Concepts You MUST Know

### 1. How Session-Based Auth Works (Step by Step)

```
┌──────────┐                         ┌──────────────┐
│  Client   │──── POST /login ──────→│   Server      │
│  (Browser)│     {user, pass}        │               │
│           │                         │  1. Validate   │
│           │                         │  2. Create      │
│           │                         │     session in  │
│           │                         │     memory/DB   │
│           │←── Set-Cookie: ────────│  3. Send cookie │
│           │    sessionId=abc123     │               │
│           │                         └──────────────┘
│           │                                         
│           │──── GET /profile ──────→ Server reads    
│           │     Cookie: sessionId=   cookie, looks    
│           │     abc123               up session in    
│           │                          memory/DB        
│           │←── 200 OK ─────────────                  
│           │    {user data}                           
│           │                                         
│           │──── POST /logout ──────→ Server deletes  
│           │     Cookie: sessionId=   session from     
│           │     abc123               memory/DB        
│           │←── Set-Cookie: ────────                  
│           │    sessionId=;Max-Age=0  (clear cookie)  
└──────────┘                                          
```

### 2. Session Storage — In-Memory Map

```js
const sessions = new Map();
const SESSION_DURATION = 15 * 60 * 1000; // 15 minutes

function createSession(user) {
    const sessionId = crypto.randomBytes(32).toString("hex");
    
    sessions.set(sessionId, {
        userId: user.id,
        username: user.name,
        expiresAt: Date.now() + SESSION_DURATION
    });
    
    return sessionId;
}
```

**Why `crypto.randomBytes(32)`?**
- 32 bytes = 256 bits of entropy. Practically unguessable.
- `toString("hex")` converts to a 64-character hex string.
- Using `Math.random()` would be predictable and insecure.

**Why a `Map` and not a plain object?**
- `Map` is designed for frequent additions/deletions (better performance).
- No prototype pollution risks (plain objects inherit from `Object.prototype`).
- `.delete()`, `.has()`, `.size` are built-in.

**Cross-Question:** *What happens when the server restarts?*
- All sessions are lost (in-memory). Users must log in again. In production, use Redis or a database for session persistence.

### 3. Cookie Handling — Set-Cookie Header

```js
// Setting cookie on login
sendJson(res, 200, { ... }, {
    "Set-Cookie": `sessionId=${sessionId}; HttpOnly; SameSite=Strict; Path=/; Max-Age=900`
});

// Clearing cookie on logout  
sendJson(res, 200, { ... }, {
    "Set-Cookie": "sessionId=; HttpOnly; SameSite=Strict; Path=/; Max-Age=0"
});
```

**Cookie flags explained:**

| Flag | Purpose |
|------|---------|
| `HttpOnly` | JavaScript can't access the cookie (`document.cookie` won't show it). Prevents XSS token theft. |
| `SameSite=Strict` | Cookie is only sent with same-origin requests. Prevents CSRF attacks. |
| `Path=/` | Cookie is valid for all paths on the domain. |
| `Max-Age=900` | Cookie expires in 900 seconds (15 minutes). |
| `Max-Age=0` | Immediately expires the cookie (logout). |

**Cross-Question:** *What's `SameSite=Lax` vs `Strict`?*
- `Strict`: Cookie never sent in cross-site requests (even clicking a link to your site from another site won't include the cookie). Most secure.
- `Lax`: Cookie sent for top-level GET navigation from other sites, but not for POST/AJAX. Good balance.
- `None`: Cookie sent with all cross-site requests (requires `Secure` flag). Used for third-party embeds.

### 4. Parsing Cookies from the Request

```js
function getSessionId(req) {
    const cookieHeader = req.headers.cookie || "";
    
    const cookie = cookieHeader
        .split(";")                              // "a=1; b=2" → ["a=1", " b=2"]
        .map(item => item.trim())                // ["a=1", "b=2"]
        .find(item => item.startsWith("sessionId="));  // "sessionId=abc123"
    
    return cookie ? cookie.slice("sessionId=".length) : null;
}
```

**Why manual parsing?**
- Node's `http` module gives you raw `Cookie` header as a string.
- Express has `cookie-parser` middleware. We're doing it manually per assignment rules.

### 5. Session Expiration Check

```js
function getSession(sessionId) {
    if (!sessionId) return null;
    
    const session = sessions.get(sessionId);
    if (!session) return null;
    
    if (Date.now() >= session.expiresAt) {
        sessions.delete(sessionId);  // Cleanup expired session
        return null;
    }
    
    return session;
}
```

**Why delete expired sessions eagerly?**
- Prevents memory leaks. Without cleanup, the Map grows indefinitely.
- A production system would also have a background cleanup job (sweep every N minutes).

---

## ⚖️ Session-Based vs JWT-Based — THE Comparison

This is the most important part of this question. **Memorize this table.**

| Aspect | Session-Based | JWT-Based |
|--------|--------------|-----------|
| **Where identity is stored** | Server-side (memory, Redis, DB) | Client-side (the token itself) |
| **What client sends** | Cookie with session ID | `Authorization: Bearer <token>` |
| **Server state** | **Stateful** — server must look up each session | **Stateless** — server only verifies the signature |
| **Scalability** | Harder — sessions must be shared across servers (sticky sessions or Redis) | Easier — any server can verify the token independently |
| **Logout** | Immediate — delete session from server | Difficult — token is valid until it expires. Need a blacklist for true revocation. |
| **Token size** | Small (just a session ID, ~64 chars) | Larger (encoded payload, ~200+ chars) |
| **Mobile apps** | Awkward — cookies are a browser concept | Natural — mobile apps easily add `Authorization` header |
| **CSRF vulnerability** | Yes — cookies are sent automatically by browsers | No — tokens must be explicitly added to headers |
| **XSS vulnerability** | Protected by `HttpOnly` cookie flag | Vulnerable if stored in localStorage |
| **Offline verification** | No — must hit the server | Yes — any service with the public key can verify |
| **User info access** | Requires DB lookup | Payload contains user info (role, ID) |

### When to Use Which?

| Use Case | Recommended |
|----------|------------|
| Traditional web app (server-rendered) | **Sessions** — cookies work seamlessly with browsers |
| SPA + API backend | **JWT** — easy to add Authorization headers from JavaScript |
| Mobile app | **JWT** — no cookie jar needed |
| Microservices | **JWT** — each service can verify independently |
| High-security (banking) | **Sessions** — instant logout, server-side control |
| Need instant revocation | **Sessions** — or JWT with blacklist |

---

## 🔒 Security Deep Dive

### Session Security Risks
1. **Session Fixation** — Attacker sets a known session ID before login. Mitigation: Regenerate session ID after login.
2. **Session Hijacking** — Attacker steals session ID (network sniffing). Mitigation: HTTPS, `Secure` cookie flag.
3. **CSRF** — Browser automatically includes cookies in cross-origin requests. Mitigation: `SameSite`, CSRF tokens.

### JWT Security Risks
1. **Token Theft** — If token is stolen (XSS, network), attacker has access until expiry. Mitigation: Short expiry, HttpOnly cookies for storage.
2. **No Revocation** — Can't invalidate a JWT before expiry without server-side state (defeats the purpose). Mitigation: Short-lived access tokens + refresh tokens.
3. **Payload Exposure** — JWT payload is base64-encoded, NOT encrypted. Anyone can decode it. Never put sensitive data in the payload.

---

## 🎯 Cross-Questions to Prepare For

| Question | Your Answer |
|----------|------------|
| Can you "invalidate" a JWT? | Not inherently. Options: 1) Short expiry (15min). 2) Token blacklist (in Redis). 3) Change the signing secret (invalidates ALL tokens). 4) Refresh token rotation. |
| What happens if two servers share sessions? | Without shared storage, Server A doesn't know about Server B's sessions. Solution: Store sessions in Redis, database, or use sticky sessions (route user to same server). |
| Why `HttpOnly` cookies? | Prevents JavaScript from reading the cookie. If an XSS attack injects malicious JavaScript, it can't steal the session cookie. |
| How do refresh tokens work? | Short-lived access token (15min) + long-lived refresh token (7 days). When access token expires, client sends refresh token to get a new access token. Refresh tokens are stored in the DB and can be revoked. |
| Is the JWT payload encrypted? | No! It's base64url-encoded (easily decoded). Never put passwords, credit cards, or sensitive data in JWTs. For encrypted tokens, use JWE (JSON Web Encryption). |
| What's `SameSite` cookie attribute? | Controls whether cookies are sent with cross-origin requests. `Strict` = never. `Lax` = only top-level GET. `None` = always (must be `Secure`). |
| What is CSRF and how do sessions enable it? | Cross-Site Request Forgery. An attacker's page makes a request to your API — the browser automatically includes your session cookie. The server thinks it's a legitimate request. |

---

## 🔧 How to Run

```bash
cd question-07-session-based-auth
node server.js
```

## 🧪 Test with curl

```bash
# Login (stores cookie)
curl -X POST http://localhost:3000/login \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "admin123"}' \
  -c cookies.txt

# Access protected route (sends cookie)
curl http://localhost:3000/profile -b cookies.txt

# Logout
curl -X POST http://localhost:3000/logout -b cookies.txt -c cookies.txt

# Try accessing after logout (401)
curl http://localhost:3000/profile -b cookies.txt
```

---

## 📚 Key Node.js Concepts Demonstrated

1. **Session Management** — In-memory session store with `Map`
2. **Cookie Handling** — `Set-Cookie` header, parsing `Cookie` header
3. **Cookie Security Flags** — `HttpOnly`, `SameSite`, `Max-Age`
4. **Secure Random Generation** — `crypto.randomBytes(32)` for session IDs
5. **Session Expiration** — TTL-based cleanup
6. **Stateful vs Stateless Auth** — Practical comparison with code
