# Deployment Guide

This project is deployed as a Docker Compose stack with MySQL, Redis, the Spring Boot backend, the React frontend served by Nginx, and phpMyAdmin for database inspection.

## Services

| Service | Image/build | Public port | Purpose |
| --- | --- | --- | --- |
| `db` | `mysql:8.0` | `3307:3306` | MySQL database |
| `redis` | `redis:7-alpine` | `6379:6379` | Redis data store |
| `backend` | `medical-platform-backend/backend.Dockerfile` | `8080:8080` | Spring Boot API |
| `frontend` | `healthcare-app-frontend/frontend.Dockerfile` | `80:80` | Nginx + React build |
| `phpmyadmin` | `phpmyadmin/phpmyadmin` | `8081:80` | MySQL administration UI |

## Prerequisites

- Docker and Docker Compose
- A root `.env` file containing backend runtime variables
- Cloudinary account for image uploads
- Stripe keys for payment checkout and webhooks
- SMTP credentials for password reset email
- Hugging Face token/model configuration if the AI endpoint is used

## Environment File

Create `.env` in the repository root. Use real values in your environment, but do not commit them.

```env
SPRING_APPLICATION_NAME=backend_med
SPRING_DATASOURCE_URL=jdbc:mysql://db:3306/dbs_prmed?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true
SPRING_DATASOURCE_USERNAME=root
SPRING_DATASOURCE_PASSWORD=change_me
SERVER_PORT=8080

JWT_SECRET=replace_with_a_long_random_secret
JWT_EXPIRATION=3600000

CLOUDINARY_CLOUD_NAME=your_cloud_name
CLOUDINARY_API_KEY=your_api_key
CLOUDINARY_API_SECRET=your_api_secret

SPRING_MAIL_HOST=smtp.gmail.com
SPRING_MAIL_PORT=587
SPRING_MAIL_USERNAME=your_email@example.com
SPRING_MAIL_PASSWORD=your_app_password

HUGGINGFACE_API_TOKEN=your_token
HUGGINGFACE_API_URL=https://api-inference.huggingface.co/models
HUGGINGFACE_API_MODEL=HuggingFaceTB/SmolLM2-1.7B-Instruct

SPRING_DATA_REDIS_HOST=redis
SPRING_DATA_REDIS_PORT=6379

STRIPE_API_KEY=sk_live_or_test_key
STRIPE_WEBHOOK_SECRET=whsec_value
STRIPE_APPOINTMENT_AMOUNT=20000
STRIPE_APPOINTMENT_CURRENCY=mad
STRIPE_SUBSCRIPTION_CURRENCY=mad
APP_FRONTEND_URL=http://localhost
```

For local Docker Compose, `SPRING_DATASOURCE_URL` must point to `db:3306`, not `localhost`.

## Build And Start

From the repository root:

```bash
docker compose up --build
```

Run in the background:

```bash
docker compose up -d --build
```

Check service status:

```bash
docker compose ps
```

Follow logs:

```bash
docker compose logs -f backend
docker compose logs -f frontend
```

Stop the stack:

```bash
docker compose down
```

Stop and remove database/Redis volumes:

```bash
docker compose down -v
```

## URLs

- Frontend: `http://localhost`
- Backend API: `http://localhost:8080`
- WebSocket queue: `ws://localhost/ws/queue` through Nginx, or `ws://localhost:8080/ws/queue` directly
- phpMyAdmin: `http://localhost:8081`
- MySQL from host tools: `localhost:3307`
- Redis from host tools: `localhost:6379`

## Frontend Runtime Routing

The frontend Docker image is a static Vite build served by Nginx.

Nginx proxies:

- `/api/` to `http://backend:8080`
- `/ws/` to `http://backend:8080`

In the current `docker-compose.yml`, the frontend build args are empty:

```yaml
args:
  VITE_API_URL: ""
  VITE_WS_URL: ""
```

Empty `VITE_API_URL` makes frontend REST calls use relative `/api/...` paths, which Nginx proxies to the backend service. For WebSockets, set `VITE_WS_URL` to an absolute browser URL for the Nginx/frontend origin:

```yaml
args:
  VITE_API_URL: ""
  VITE_WS_URL: "ws://localhost"
```

For a separate frontend host or CDN, set these build args to public backend URLs such as:

```yaml
args:
  VITE_API_URL: "https://api.example.com"
  VITE_WS_URL: "wss://api.example.com"
```

Then rebuild the frontend image.

## Database Persistence

Docker volumes persist data between restarts:

- `mysql_data` stores MySQL data
- `redis_data` stores Redis append-only data

Use `docker compose down -v` only when you intentionally want to erase local persisted data.

## CI/CD

Each app folder has its own GitHub Actions workflow:

- `medical-platform-backend/.github/workflows/main.yml` runs Maven verification and builds/publishes a backend image.
- `healthcare-app-frontend/.github/workflows/main.yml` runs lint, Vitest coverage, Vite build, and builds/publishes a frontend image.

The workflows publish Docker images to GitHub Container Registry (`ghcr.io`) on non-pull-request events.

## Production Checklist

- Replace default database credentials.
- Rotate any secrets that were ever committed or shared.
- Use production Cloudinary, Stripe, SMTP, and JWT secrets.
- Set `APP_FRONTEND_URL` to the deployed frontend origin.
- Update backend CORS origins in `SecurityConfig` for the production frontend domain.
- Replace or disable default admin initialization before exposing the app.
- Configure HTTPS and use `wss://` for WebSocket traffic.
- Configure a real Stripe webhook secret and point Stripe webhooks to `/api/payments/webhook`.
- Back up MySQL volumes or use a managed database.
- Avoid exposing phpMyAdmin publicly in production.
