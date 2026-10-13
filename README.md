# FastAPI Microservices Platform with Nginx API Gateway

A containerized microservices backend built with **FastAPI** and **PostgreSQL**, with an **Nginx** reverse proxy acting as the single entry point for all services.

---

## Overview

The system is split into three independent services behind one gateway:

| Component | Responsibility |
|-----------|----------------|
| **User Service** | Registration, login, JWT issuance, profile lookup |
| **Order Service** | Order creation and lifecycle, talks to other services |
| **Payment Service** | Payment processing, status tracking, refunds |
| **Nginx Gateway** | Routing, rate limiting, TLS termination, docs routing |
| **PostgreSQL** | One database per service (database-per-service pattern) |
| **Docker Compose** | Orchestrates everything with health checks |

---

## Tech Stack

- **Backend:** Python, FastAPI, Pydantic, SQLAlchemy
- **Auth:** JWT (bearer tokens)
- **Data:** PostgreSQL
- **Gateway:** Nginx (HTTP + HTTPS)
- **DevOps:** Docker, Docker Compose

---

## Getting Started

### Requirements

- Docker and Docker Compose
- Git
- Python 3.9+ (only if you want to run a service outside Docker)

### Setup

```bash
# 1. Clone this repository
git clone https://github.com/<your-username>/<your-repo-name>.git
cd <your-repo-name>

# 2. Create your environment file and edit the values
cp .env.example .env

# 3. Generate a self-signed certificate for HTTPS
chmod +x scripts/generate-ssl.sh
./scripts/generate-ssl.sh

# 4. Build and start the stack
docker compose up --build
```

Once the containers are healthy:

| What | URL |
|------|-----|
| Gateway (HTTP) | http://localhost |
| Gateway (HTTPS) | https://localhost |
| Users | http://localhost/users/ |
| Orders | http://localhost/orders/ |
| Payments | http://localhost/payments/ |

### HTTPS Certificates

Certificates are gitignored so private keys never get committed. Pick one option:

**Self-signed (development)**

```bash
mkdir -p nginx/ssl
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nginx/ssl/nginx.key \
  -out nginx/ssl/nginx.crt \
  -subj "/C=US/ST=State/L=City/O=Organization/CN=localhost"
```

**Helper script:** `./scripts/generate-ssl.sh`

**Your own certificate:** place `nginx.crt` and `nginx.key` inside `nginx/ssl/`.

---

## API Reference

### Users — `/users/`

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | `/users/register` | Create an account (email, username, password; full name optional) | No |
| POST | `/users/login` | Log in with email **or** username, returns a JWT | No |
| GET | `/users/me` | Current user's profile | Yes |
| GET | `/users/{user_id}` | Fetch a user by ID | Yes |
| GET | `/users/` | List users | Yes |

### Orders — `/orders/`

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | `/orders/` | Place an order | Yes |
| GET | `/orders/{order_id}` | Order details | Yes |
| GET | `/orders/` | Orders of the current user | Yes |
| PATCH | `/orders/{order_id}/status` | Update order status | Yes |

### Payments — `/payments/`

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | `/payments/` | Process a payment (token must belong to the paying user) | Yes |
| GET | `/payments/{payment_id}` | Payment status | Yes |
| GET | `/payments/` | Payments of the current user | Yes |
| POST | `/payments/{payment_id}/refund` | Refund a payment | Yes |

### Interactive Docs

- Users: http://localhost/users/docs
- Orders: http://localhost/orders/docs
- Payments: http://localhost/payments/docs
- Health: http://localhost/health

---

## Try It Out

```bash
# Gateway health
curl http://localhost/health

# Register
curl -X POST "http://localhost/users/register" \
  -H "Content-Type: application/json" \
  -d '{"username": "testuser", "email": "test@example.com", "password": "testpass123"}'

# Log in and copy the token from the response
curl -X POST "http://localhost/users/login" \
  -H "Content-Type: application/json" \
  -d '{"username": "testuser", "password": "testpass123"}'

# Place an order using the token
curl -X POST "http://localhost/orders/" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <YOUR_JWT_TOKEN>" \
  -d '{
    "items": [
      {"product_name": "Product 1", "quantity": 2, "price": 29.99},
      {"product_name": "Product 2", "quantity": 1, "price": 49.99}
    ]
  }'
```

---

## Project Layout

```
.
├── services/
│   ├── user-service/
│   │   ├── app/
│   │   │   ├── main.py        # FastAPI app + custom Swagger UI
│   │   │   ├── models.py      # SQLAlchemy models
│   │   │   ├── schemas.py     # Pydantic schemas
│   │   │   ├── auth.py        # JWT logic
│   │   │   ├── database.py    # DB session/engine
│   │   │   └── config.py      # Settings
│   │   ├── Dockerfile
│   │   └── requirements.txt
│   ├── order-service/         # Same layout as user-service
│   └── payment-service/       # Same layout as user-service
├── nginx/
│   ├── nginx.conf             # Main gateway config
│   ├── conf.d/locations.conf  # Route definitions
│   └── ssl/                   # Certificates (gitignored)
├── scripts/generate-ssl.sh
├── docker-compose.yml
├── .env.example
├── init.sql                   # Creates the per-service databases
└── README.md
```

---

## Configuration

Every service reads its settings from environment variables:

| Variable | Purpose |
|----------|---------|
| `DATABASE_URL` | PostgreSQL connection string |
| `JWT_SECRET` | Key used to sign tokens |
| `LOG_LEVEL` | `DEBUG`, `INFO`, `WARNING` or `ERROR` |

### Gateway Behaviour

- **Routing:** clean prefixes `/users/`, `/orders/`, `/payments/`
- **Docs routing:** exact-match routes for each service's `/docs` and `openapi.json`, so Swagger UI loads the correct schema even behind the proxy
- **Rate limiting:** 10 requests/second per client with a burst of 20
- **TLS:** HTTP and HTTPS listeners
- **Security headers:** XSS protection and MIME-sniffing prevention
- **Caching headers:** prevents stale documentation in browsers
- **Errors:** consistent JSON error responses
- **Health checks:** gateway and service readiness endpoints

---

## 💻 Local Development

```bash
# Start the database dependency (PostgreSQL is mapped to host port 5434)
docker compose up -d postgres

# Run one service directly
cd services/user-service
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt

DATABASE_URL=postgresql://postgres:<your-password>@localhost:5434/user_db \
uvicorn app.main:app --reload --port 8001
```

**Verification:** use the curl examples above against the running Compose stack.

---

## Security Notes

- JWT-protected endpoints with per-user data access
- Rate limiting and TLS at the gateway
- Pydantic validation on all request bodies
- SQLAlchemy ORM to avoid raw SQL injection risks
- Secrets loaded from environment variables, never hardcoded

**Before going to production:**

- Replace self-signed certificates with trusted ones
- Use a secrets manager (Docker secrets, Vault, etc.)
- Restrict CORS to your own domain
- Tune PostgreSQL connection pooling and Nginx worker processes
- Consider database caching and read replicas for scale

---

## Concepts Covered

- Service decomposition and database-per-service
- API gateway pattern with Nginx
- Inter-service communication
- JWT authentication in FastAPI
- Swagger UI behind a reverse proxy
- Containerization and health-checked orchestration with Docker Compose


---

## License

MIT. See [LICENSE](LICENSE).
