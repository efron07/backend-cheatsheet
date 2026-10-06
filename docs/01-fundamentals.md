# 1. Backend fundamentals

## 1.1 What a backend does

**What it is:** The backend is the part of the system users never see. It receives requests from the frontend (or a mobile app, or another server), applies the **business rules**, reads and writes the **database**, talks to **external services** (M-Pesa, SMS, email), and sends back a response.

**PharmaCare example:** when a cashier presses "Complete sale", the backend checks the user is logged in and allowed to sell, checks stock, picks the batch that expires first, deducts stock, records the sale, starts the M-Pesa payment, and returns the receipt. The frontend cannot be trusted to do any of this: anyone can change browser code.

**Backend responsibilities:**

- **Data:** storing, querying and protecting it.
- **Business logic:** rules like "never sell expired stock" and "cashiers cannot change prices".
- **Security:** authentication, authorisation, validation.
- **Integration:** payment providers, SMS, email, other systems.
- **Performance and reliability:** caching, queues, scaling, monitoring.

**Backend language choices:** JavaScript/TypeScript (Node.js), PHP (Laravel), Python (Django, FastAPI), Go, Java/Kotlin (Spring), C# (.NET), Rust. This sheet uses **Node.js + TypeScript**, so frontend and backend share one language and even share types and Zod schemas.

## 1.2 How the internet carries a request

1. The client looks up `api.pharmacare.co.tz` in **DNS** and gets an IP address.
2. It opens a **TCP** connection (or **QUIC** for HTTP/3) and a **TLS** handshake for HTTPS.
3. It sends an **HTTP request**; a load balancer or reverse proxy (Nginx) forwards it to your Node.js process.
4. Your code runs and returns an **HTTP response**.

**Key terms:** IP address, port (`3000` for your API, `443` for HTTPS, `5432` Postgres, `27017` MongoDB, `6379` Redis), domain, hosting, reverse proxy, load balancer, CDN.

## 1.3 HTTP in depth

**An HTTP request:**

```http
POST /v1/sales HTTP/1.1
Host: api.pharmacare.co.tz
Content-Type: application/json
Accept: application/json
Authorization: Bearer eyJhbGciOi...
Idempotency-Key: 7f9c2e7a-1b3d-4f0a-9d2e-8a1c5b6e7d90

{ "paymentMethod": "cash", "items": [{ "medicineId": 7, "qty": 2 }] }
```

**An HTTP response:**

```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: /v1/sales/1052

{ "data": { "id": 1052, "receiptNo": "RCP-20261006-0041", "total": 1000 } }
```

**Methods:**

| Method | Meaning | Safe (no changes)? | Idempotent (same result if repeated)? |
| --- | --- | --- | --- |
| GET | Read | Yes | Yes |
| POST | Create / run an action | No | No |
| PUT | Replace a whole resource | No | Yes |
| PATCH | Change part of a resource | No | Not necessarily |
| DELETE | Remove | No | Yes |
| HEAD / OPTIONS | Headers only / allowed methods (CORS preflight) | Yes | Yes |

**Status codes:**

| Code | Use it when |
| --- | --- |
| 200 OK | Successful read or update |
| 201 Created | Resource created (add a `Location` header) |
| 204 No Content | Success with nothing to return (delete) |
| 301 / 308 | Permanently moved |
| 304 Not Modified | Client's cached copy is still valid (ETag) |
| 400 Bad Request | Malformed request (invalid JSON) |
| 401 Unauthorized | Not logged in / bad token |
| 403 Forbidden | Logged in but not allowed |
| 404 Not Found | Resource does not exist |
| 409 Conflict | Clashes with current state (duplicate batch number, not enough stock) |
| 422 Unprocessable Content | Validation failed |
| 429 Too Many Requests | Rate limit hit |
| 500 Internal Server Error | A bug (never expose details) |
| 502 / 503 / 504 | Upstream failed / unavailable / timed out |

**Important headers:** `Content-Type`, `Accept`, `Authorization`, `Cookie` / `Set-Cookie`, `Cache-Control`, `ETag` / `If-None-Match`, `Location`, `Retry-After`, `X-Request-Id`, CORS headers (`Access-Control-Allow-Origin`).

**HTTP is stateless:** each request stands alone. State (who is logged in) travels in a cookie or token on every request, or lives in the database/Redis.

**Versions:** HTTP/1.1 (text, one request at a time per connection), HTTP/2 (multiplexed, binary), HTTP/3 (over QUIC, faster on bad mobile networks). Your reverse proxy or host handles this; Node usually speaks HTTP/1.1 behind it.

## 1.4 The terminal and operating system basics

Backend work happens on Linux servers. You need to be comfortable with:

```bash
pwd; ls -la; cd /var/www/pharmacare-api       # where am I, list, move
mkdir logs; touch .env; cp a b; mv a b; rm file  # files
cat .env; less app.log; tail -f app.log          # read files, follow a log live
grep -R "DATABASE_URL" src/                       # search text
chmod 600 .env; chown app:app .env               # permissions
ps aux | grep node; kill -SIGTERM <pid>          # processes
top / htop; free -h; df -h                       # CPU, memory, disk
curl -i https://api.pharmacare.co.tz/health      # call an HTTP endpoint
ss -tlnp                                         # which ports are listening
ssh deploy@server; scp file deploy@server:/tmp   # remote access and copying
export NODE_ENV=production; env                  # environment variables
```

**OS concepts that matter for a backend:** processes and threads, memory, file descriptors (open connections), signals (`SIGTERM` = please shut down), environment variables, permissions, networking (ports, sockets).
