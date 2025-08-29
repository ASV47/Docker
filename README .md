
# Flask + MySQL + Redis (Docker Compose)

A production‑ready, containerized Flask application stack with MySQL and Redis. This repository includes a minimal app, reproducible local environment, and deployment‑oriented guidance.

---

## Status & Tech Stack
![Python](https://img.shields.io/badge/Python-3.9%2F3.11-blue)
![Flask](https://img.shields.io/badge/Flask-2%2B-green)
![Docker](https://img.shields.io/badge/Docker-Compose-informational)
![License](https://img.shields.io/badge/License-MIT-lightgrey)

---

## Overview
This project demonstrates a clean separation of concerns:
- **flask_app** – Python/Flask API
- **mysql_server** – MySQL 5.7 database
- **redis_server** – Redis cache (internal network only)

The stack is designed to be simple to run locally yet aligned with production practices (12‑factor configs, immutable images, no host‑exposed Redis by default, optional volumes).

---

## Architecture (logical)
```
[ Client ]  -->  http://localhost:8080
                   |
                   v
             [flask_app :6000]
                |         |
     SQLAlchemy / PyMySQL  \--> Redis client
                |                  
          [mysql_server:3306]   [redis_server:6379]
```
- Containers communicate over the Compose network using service names as hostnames.
- Only the Flask service exposes a host port (8080 → 6000).

---

## Prerequisites
- Docker 24+ and Docker Compose v2+
- Make (optional) for convenience targets

---

## Quick start
```bash
git clone https://github.com/USERNAME/REPO_NAME.git
cd REPO_NAME
cp .env.example .env   # or create .env as below
docker compose up --build
# App: http://localhost:8080
```

---

## Configuration
Environment variables are loaded from `.env` at the repository root:

```env
# MySQL
MYSQL_ROOT_PASSWORD=root
MYSQL_DATABASE=mydb
MYSQL_USER=admin
MYSQL_PASSWORD=1234

# Flask App
APP_PORT=6000
FLASK_ENV=development
```

> Do **not** commit real credentials. Keep `.env` out of version control.

---

## Compose services
`docker-compose.yml` (excerpt):
```yaml
services:
  flask_app:
    build: .
    ports:
      - "8080:6000"
    environment:
      DATABASE_URL: mysql+pymysql://${MYSQL_USER}:${MYSQL_PASSWORD}@mysql_server:3306/${MYSQL_DATABASE}
      REDIS_URL:    redis://redis_server:6379/0
    depends_on:
      mysql_server:
        condition: service_healthy
      redis_server:
        condition: service_started

  mysql_server:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    ports:
      - "3307:3306"
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1", "-p${MYSQL_ROOT_PASSWORD}"]
      interval: 5s
      timeout: 3s
      retries: 15

  redis_server:
    image: redis:alpine
    # no host ports exposed by default
```

> Optional data persistence:
```yaml
  mysql_server:
    volumes:
      - db_data:/var/lib/mysql

volumes:
  db_data:
```

---

## Project structure
```
.
├── app.py                 # Flask entrypoint (or app/app.py)
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
├── .env                   # local settings (not committed)
└── README.md
```

---

## Local development
- Start stack:
  ```bash
  docker compose up --build
  ```
- Follow logs:
  ```bash
  docker compose logs -f flask_app
  ```
- Exec into the app container:
  ```bash
  docker exec -it $(docker ps -qf "name=flask_app") sh
  ```
- Stop & clean:
  ```bash
  docker compose down -v
  ```

Health endpoints (if included):
- `GET /` – liveness check
- `GET /check/mysql` – DB connectivity
- `GET /check/redis` – Redis connectivity

---

## Production notes
- Use a WSGI server (e.g., **gunicorn**) instead of Flask’s dev server.
- Example Dockerfile CMD:
  ```dockerfile
  RUN pip install --no-cache-dir gunicorn
  CMD ["gunicorn","-w","2","-b","0.0.0.0:6000","app:app"]
  ```
- Externalize secrets (e.g., Docker/Kubernetes secrets or a vault).
- Pin image tags and library versions. Enable health checks and observability in your runtime.

---

## Troubleshooting
- **Port already allocated** (e.g., Redis 6379): do not expose Redis to host; remove `ports:` from `redis_server` or change the host port.
- **App container unhealthy**: ensure the app listens on `0.0.0.0:${APP_PORT}` and the CMD path to `app.py` is correct.
- **DB access denied**: confirm credentials/host `mysql_server` and port `3306`. Recreate DB volume if necessary.

---

## License
ASV47 ;)
