# Docker & Deployment Guide

## Overview

This guide covers Docker setup, containerization, and deployment strategies for the payment gateway system.

## Prerequisites

- Docker 20.10+
- Docker Compose 2.0+
- Docker Hub account (for image registry)

## Quick Start

### One-Command Deployment

```bash
docker-compose up -d
```

This starts:
- PostgreSQL database (port 5432)
- Backend API (port 5000)
- Frontend (port 3000)

## Docker Architecture

```
┌─────────────────────────────────────┐
│     Docker Network: payment-gw      │
├─────────────────────────────────────┤
│                                     │
│  ┌──────────────┐ ┌──────────────┐ │
│  │   Frontend   │ │   Backend    │ │
│  │  (React)     │ │  (Node.js)   │ │
│  │  Port: 3000  │ │  Port: 5000  │ │
│  └──────────────┘ └──────────────┘ │
│         │              │            │
│         └──────────────┬────────────┘
│                        │            │
│                 ┌──────────────┐    │
│                 │  PostgreSQL  │    │
│                 │ Port: 5432   │    │
│                 └──────────────┘    │
│                                     │
└─────────────────────────────────────┘
```

## Docker Compose Configuration

### docker-compose.yml

```yaml
version: '3.8'

services:
  # PostgreSQL Database
  db:
    image: postgres:15-alpine
    container_name: payment-gw-db
    environment:
      POSTGRES_USER: ${DB_USER:-postgres}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-postgres}
      POSTGRES_DB: ${DB_NAME:-payment_gateway}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - payment-network

  # Backend API
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: payment-gw-api
    ports:
      - "5000:5000"
    environment:
      NODE_ENV: ${NODE_ENV:-development}
      PORT: 5000
      DATABASE_URL: postgresql://${DB_USER}:${DB_PASSWORD}@db:5432/${DB_NAME}
      JWT_SECRET: ${JWT_SECRET:-your_secret_key}
      API_KEY_SECRET: ${API_KEY_SECRET:-your_api_secret}
      FRONTEND_URL: http://localhost:3000
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - ./backend/src:/app/src
    networks:
      - payment-network
    restart: unless-stopped

  # Frontend Application
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: payment-gw-ui
    ports:
      - "3000:3000"
    environment:
      REACT_APP_API_URL: http://localhost:5000
      REACT_APP_ENVIRONMENT: development
    depends_on:
      - backend
    volumes:
      - ./frontend/src:/app/src
    networks:
      - payment-network
    restart: unless-stopped

volumes:
  postgres_data:

networks:
  payment-network:
    driver: bridge
```

## Backend Dockerfile

### backend/Dockerfile

```dockerfile
# Build stage
FROM node:18-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

# Runtime stage
FROM node:18-alpine

WORKDIR /app

# Install dumb-init for proper signal handling
RUN apk add --no-cache dumb-init

COPY --from=builder /app/node_modules ./node_modules
COPY . .

EXPOSE 5000

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:5000/health', (r) => {if (r.statusCode !== 200) throw new Error(r.statusCode)})"

ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "server.js"]
```

## Frontend Dockerfile

### frontend/Dockerfile

```dockerfile
# Build stage
FROM node:18-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

RUN npm run build

# Nginx stage
FROM nginx:alpine

COPY --from=builder /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 3000

CMD ["nginx", "-g", "daemon off;"]
```

### nginx.conf

```nginx
server {
    listen 3000;
    server_name localhost;

    root /usr/share/nginx/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api {
        proxy_pass http://backend:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    error_page 404 /index.html;
}
```

## Environment Setup

### .env File

```bash
# Database
DB_USER=postgres
DB_PASSWORD=secure_password_here
DB_NAME=payment_gateway

# Backend
NODE_ENV=development
JWT_SECRET=your_jwt_secret_key_here
API_KEY_SECRET=your_api_key_secret_here

# Frontend
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ENVIRONMENT=development
```

## Docker Commands

### Build Images

```bash
# Build all services
docker-compose build

# Build specific service
docker-compose build backend
docker-compose build frontend
```

### Run Services

```bash
# Start all services in background
docker-compose up -d

# Start with logs
docker-compose up

# Stop all services
docker-compose down

# Remove volumes (data)
docker-compose down -v
```

### View Logs

```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f backend
docker-compose logs -f frontend
docker-compose logs -f db

# Last 100 lines
docker-compose logs --tail=100
```

### Execute Commands

```bash
# Backend shell
docker-compose exec backend sh

# Run migrations
docker-compose exec backend npm run migrate

# Database shell
docker-compose exec db psql -U postgres -d payment_gateway
```

## Production Deployment

### Build for Production

```bash
# Build optimized images
docker-compose -f docker-compose.prod.yml build --no-cache

# Push to registry
docker tag payment-gw-api:latest your-registry/payment-gw-api:latest
docker push your-registry/payment-gw-api:latest
```

### Production Docker Compose

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  db:
    image: postgres:15-alpine
    restart: always
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - postgres_prod_data:/var/lib/postgresql/data
    networks:
      - payment-network

  backend:
    image: your-registry/payment-gw-api:latest
    restart: always
    environment:
      NODE_ENV: production
      DATABASE_URL: postgresql://${DB_USER}:${DB_PASSWORD}@db:5432/${DB_NAME}
      JWT_SECRET: ${JWT_SECRET}
      API_KEY_SECRET: ${API_KEY_SECRET}
    depends_on:
      - db
    networks:
      - payment-network

  frontend:
    image: your-registry/payment-gw-ui:latest
    restart: always
    networks:
      - payment-network

volumes:
  postgres_prod_data:

networks:
  payment-network:
```

## AWS ECS Deployment

### Task Definition (Backend)

```json
{
  "family": "payment-gw-backend",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "containerDefinitions": [
    {
      "name": "backend",
      "image": "your-account.dkr.ecr.region.amazonaws.com/payment-gw-api:latest",
      "portMappings": [
        {
          "containerPort": 5000,
          "hostPort": 5000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "NODE_ENV",
          "value": "production"
        }
      ],
      "secrets": [
        {
          "name": "DATABASE_URL",
          "valueFrom": "arn:aws:secretsmanager:region:account:secret:payment-gw-db-url"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/payment-gw-backend",
          "awslogs-region": "region",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

## Health Checks

### Backend Health Endpoint

```javascript
// src/routes/health.js
router.get('/health', (req, res) => {
  res.json({ status: 'ok', timestamp: new Date() });
});
```

## Security Best Practices

1. **Use multi-stage builds** to reduce image size
2. **Don't run as root** - use specific user
3. **Scan images** - `docker scan your-image`
4. **Use secrets management** - AWS Secrets Manager, HashiCorp Vault
5. **Enable TLS** - Nginx reverse proxy with SSL
6. **Resource limits** - Set memory/CPU limits

## Monitoring & Logging

### Container Logs

```bash
# Real-time logs
docker-compose logs -f --tail=50

# Export logs
docker-compose logs > application.log
```

### Docker Stats

```bash
# Resource usage
docker stats

# Specific container
docker stats payment-gw-api
```

## Troubleshooting

### Port Already in Use

```bash
# Change ports in docker-compose.yml
ports:
  - "8000:3000"  # Map 8000 to 3000

# Or stop conflicting container
docker-compose down
lsof -i :3000  # Find process
```

### Database Connection Issues

```bash
# Check if db is ready
docker-compose exec db pg_isready

# View db logs
docker-compose logs db
```

### Out of Disk Space

```bash
# Cleanup unused images/volumes
docker system prune -a

# Remove specific volume
docker volume rm payment-gw_postgres_data
```

## Performance Optimization

1. **Use Alpine images** - Smaller base images
2. **Layer caching** - Order Dockerfile commands by change frequency
3. **Limit image layers** - Combine RUN commands
4. **Clean up** - Remove build artifacts in image

## References

- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose](https://docs.docker.com/compose/)
- [Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [AWS ECS](https://docs.aws.amazon.com/ecs/)
