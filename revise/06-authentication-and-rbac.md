# Question 06 — Authentication and Role-Based Authorization

## 📋 What Was Asked

Implement JWT authentication and role-based access control (RBAC). Create a login endpoint, protect routes, enforce roles (admin/user), and handle token edge cases (missing, invalid, expired).

---

## 🏗️ Architecture & File Structure

```
question-06-auth-and-RBAC/
├── server.js                    ← Entry point, JWT_SECRET validation
├── .env                         ← JWT_SECRET and JWT_EXPIRES_IN
├── .gitignore                   ← Keeps .env out of version control
├── routes/
│   ├── index.js                 ← Master router — dispatches to sub-routers
│   ├── authRoutes.js            ← POST /login
│   └── userRoutes.js            ← CRUD routes (protected)
├── middleware/
│   └── auth.js                  ← authenticateToken() + authorizeRoles()
├── services/
│   ├── authService.js           ← Login logic (bcrypt + JWT)
│   └── userService.js           ← User CRUD (same as Q5)
├── utils/
│   └── httpUtils.js             ← sendJson + readJsonBody
└── data/
    ├── accounts.json            ← Login credentials (email, passwordHash, role)
    └── users.json               ← User records for CRUD
```

### Why Separate `accounts.json` and `users.json`?

- `accounts.json` = **authentication data** (credentials, roles). This is "who can log in."
- `users.json` = **application data** (the resources being managed). This is "what the API manages."
- Mixing them would violate separation of concerns. In a real app, these would be separate database tables.

---

## 🔑 Core Concepts You MUST Know

### 1. JWT (JSON Web Token) — How It Works

A JWT has three parts separated by dots: `header.payload.signature`

```
eyJhbGciOiJIUzI1NiJ9.eyJyb2xlIjoiYWRtaW4iLCJzdWIiOiJhZG1pbkFjYyJ9.abc123signature
│                      │                                                      │
│  Header              │  Payload                                             │  Signature
│  {"alg":"HS256"}     │  {"role":"admin","sub":"adminAcc","exp":1234567890}   │  HMAC-SHA256
```

**The flow:**
```
1. Client sends POST /login with email + password
2. Server verifies credentials against accounts.json (using bcrypt)
3. Server creates JWT with { role, sub (user ID), exp (expiry) }
4. Server signs the JWT with JWT_SECRET
5. Client stores the token and sends it in every future request:
   Authorization: Bearer <token>
6. Server verifies the token's signature on each protected request
```

**Cross-Questions:**

- **Q: Where should the client store the JWT?**
  - **localStorage** — easy but vulnerable to XSS attacks.
  - **HttpOnly cookie** — can't be accessed by JavaScript, safer against XSS. Vulnerable to CSRF (mitigate with SameSite).
  - **Memory (variable)** — safest against XSS, but lost on page refresh.
  - For APIs consumed by mobile apps, localStorage or secure storage is fine.

- **Q: Why not just use sessions?**
  - JWTs are **stateless** — the server doesn't store anything. Scales horizontally without shared session storage. Perfect for microservices and mobile apps. (We compare in Q7.)

- **Q: What if the JWT_SECRET leaks?**
  - Anyone can forge valid tokens. Immediately rotate the secret (all existing tokens become invalid). In production, use asymmetric keys (RS256) so only the server can sign, but anyone can verify.

### 2. Token Creation

```js
const token = jwt.sign(
    { role: account.role },           // Payload (claims)
    process.env.JWT_SECRET,            // Secret key
    {
        subject: account.id,           // Standard claim: who this token is for
        expiresIn: process.env.JWT_EXPIRES_IN || '1h'  // Auto-expiry
    }
);
```

**Why put `role` in the payload?**
- So we can check permissions without a database lookup on every request.
- The role is baked into the token. If the user's role changes, the old token still has the old role (a tradeoff — use short expiry to mitigate).

**Why use `subject` instead of custom `userId`?**
- `sub` is a standard JWT claim (RFC 7519). Using standards makes your tokens interoperable with other JWT libraries/services.

### 3. Token Verification (Authentication Middleware)

```js
function authenticateToken(req, res) {
    const authHeader = req.headers.authorization;
    const match = typeof authHeader === 'string'
        ? authHeader.trim().match(/^Bearer\s+(\S+)$/i)
        : null;

    if (!match) {
        sendJson(res, 401, { message: 'Authentication token required' });
        return null;
    }

    try {
        return jwt.verify(match[1], process.env.JWT_SECRET);
    } catch (error) {
        sendJson(res, 401, {
            message: error.name === 'TokenExpiredError'
                ? 'Token expired'
                : 'Invalid token'
        });
        return null;
    }
}
```

**The regex `^Bearer\s+(\S+)$/i` explained:**
```
^Bearer   → Must start with "Bearer"
\s+       → One or more spaces
(\S+)     → Capture the token (non-whitespace characters)
$         → Must end here
/i        → Case-insensitive ("bearer" also works)
```

**Why case-insensitive?**
- RFC 6750 specifies "Bearer" but some clients send "bearer." Being flexible avoids unnecessary failures.

**Error differentiation:**
| Error | Status | Message |
|-------|--------|---------|
| No `Authorization` header | 401 | Token required |
| Invalid/tampered token | 401 | Invalid token |
| Expired token | 401 | Token expired |

All return 401 but with different messages — helps the client understand what went wrong.

### 4. Role-Based Access Control (RBAC)

```js
function authorizeRoles(res, user, ...roles) {
    if (!roles.includes(user.role)) {
        sendJson(res, 403, {
            message: 'You do not have permission to perform this operation'
        });
        return false;
    }
    return true;
}
```

**Usage:**
```js
// Only admins can create users
if (method === 'POST' && pathname === '/users') {
    if (!authorizeRoles(res, loggedInUser, 'admin')) {
        return true;  // Response already sent
    }
    // ... create user
}
```

**Why rest parameter `...roles`?**
- Flexibility: `authorizeRoles(res, user, 'admin')` or `authorizeRoles(res, user, 'admin', 'moderator')`.

**401 vs 403 — Know the Difference!**

| Code | Name | Meaning |
|------|------|---------|
| **401** Unauthorized | "I don't know who you are" | Missing or invalid credentials |
| **403** Forbidden | "I know who you are, but you can't do this" | Valid credentials, insufficient permissions |

### 5. Password Hashing with bcrypt

```js
const bcrypt = require('bcryptjs');

// During account creation (not shown in code, done beforehand):
const hash = await bcrypt.hash('plainPassword', 10);

// During login:
const isValid = await bcrypt.compare(password, account.passwordHash);
```

**Why bcrypt?**
- **Salting** — Each hash includes a unique salt, so identical passwords produce different hashes.
- **Adaptive cost** — The `10` (work factor) makes it intentionally slow (~100ms). This is a feature — it makes brute-force attacks impractical.
- **No reversibility** — You can't get the password back from the hash.

**Cross-Questions:**
- **Q: Why `bcryptjs` and not `bcrypt`?**
  - `bcryptjs` is a pure JavaScript implementation — no native C++ compilation needed. `bcrypt` (native) is faster but requires `node-gyp` and C++ build tools. For this project, `bcryptjs` is simpler.

- **Q: What's the cost factor (10)?**
  - The number of rounds is `2^cost`. Cost 10 = 1,024 rounds. Higher = slower = more secure but more CPU usage. 10-12 is the standard recommendation.

### 6. Environment Variables

```env
JWT_SECRET = StringWhichWillBeRandom
JWT_EXPIRES_IN=1h
```

```js
require('dotenv').config();

if (!process.env.JWT_SECRET) {
    console.error('JWT_SECRET is missing from .env');
    process.exit(1);
}
```

**Why `.env` and not hardcoded?**
- Secrets in source code get committed to Git → exposed.
- `.env` is in `.gitignore` — never committed.
- Different environments (dev, staging, production) need different secrets.

---

## 🛡️ Permission Matrix

| Endpoint | Method | admin | user |
|----------|--------|-------|------|
| `/login` | POST | ✅ (public) | ✅ (public) |
| `/users` | GET | ✅ | ✅ |
| `/users` | POST | ✅ | ❌ (403) |
| `/users/:id` | GET | ✅ | ✅ |
| `/users/:id` | PUT | ✅ | ❌ (403) |
| `/users/:id` | DELETE | ✅ | ❌ (403) |

**Design decision:** Read operations are available to any authenticated user. Write operations (create, update, delete) require admin role.

---

## 🎯 Cross-Questions to Prepare For

| Question | Your Answer |
|----------|------------|
| How does JWT verification work without a database call? | The signature is a HMAC of `header.payload` using the secret. `jwt.verify()` recalculates the signature and compares. If they match, the payload hasn't been tampered with. |
| Can a JWT be "revoked"? | Not inherently — JWTs are stateless. You'd need a blacklist (in Redis/DB) to track revoked tokens. Or use short expiry + refresh tokens. |
| What are refresh tokens? | Long-lived tokens used to get new access tokens. Stored securely, checked against a database. Allows revoking access without short-lived JWTs. |
| What's in the JWT payload? | `role`, `sub` (account ID), `iat` (issued at), `exp` (expiration). Keep payloads small — they're sent with every request. |
| Why not store the password in plain text? | If the database is breached, all passwords are exposed. Hashing is a one-way function — passwords can be verified but not recovered. |
| What's the difference between HS256 and RS256? | HS256: symmetric — same secret for signing and verifying. RS256: asymmetric — private key signs, public key verifies. RS256 is better for distributed systems. |
| How would you implement token refresh? | Issue two tokens: short-lived access token (15min) + long-lived refresh token (7 days). Client uses refresh token to get new access tokens. |

---

## 🔧 How to Run

```bash
cd question-06-auth-and-RBAC
node server.js
```

## 🧪 Test with curl

```bash
# Login as admin
curl -X POST http://localhost:3000/login \
  -H "Content-Type: application/json" \
  -d '{"email": "admin@example.com", "password": "admin123"}'

# Use the returned token for protected routes
curl http://localhost:3000/users \
  -H "Authorization: Bearer <token>"

# Try admin-only operation as regular user (403)
curl -X POST http://localhost:3000/users \
  -H "Authorization: Bearer <user-token>" \
  -H "Content-Type: application/json" \
  -d '{"name": "New", "email": "new@test.com"}'

# No token (401)
curl http://localhost:3000/users
```

---

## 📚 Key Node.js Concepts Demonstrated

1. **JWT Creation & Verification** — `jsonwebtoken` library, claims, expiry
2. **bcrypt Password Hashing** — Salting, cost factor, `compare()`
3. **Middleware Pattern** — Authentication/authorization checks before route handlers
4. **RBAC** — Role-based access control with `admin` and `user` roles
5. **Environment Variables** — `dotenv`, keeping secrets out of source code
6. **Error Differentiation** — 401 (unauthenticated) vs 403 (unauthorized)
7. **Bearer Token Parsing** — Regex extraction from `Authorization` header
