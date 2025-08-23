# Docker Deployment Configuration

## 1. Overview

This document provides comprehensive Docker deployment configuration for the Chat UI application with Supabase integration. The containerized setup eliminates the need for local MongoDB and leverages Supabase as a complete backend-as-a-service solution.

## 2. Container Architecture

### 2.1 Single Container Approach

With Supabase integration, we use a simplified single-container architecture:

```mermaid
graph TD
    A[Docker Container] --> B[Node.js Runtime]
    B --> C[SvelteKit Application]
    C --> D[Supabase Client SDK]
    D --> E[External Supabase Service]
    
    subgraph "Container Environment"
        A
        B
        C
        F[Environment Variables]
        G[Application Files]
        H[Node Modules]
    end
    
    subgraph "External Services"
        E
        I[AI Model APIs]
        J[Web Search APIs]
    end
    
    C --> I
    C --> J
```

### 2.2 Benefits of Supabase Integration

- **No Database Container**: Eliminates MongoDB container complexity
- **Reduced Resource Usage**: Lower memory and CPU requirements
- **Simplified Networking**: No inter-container communication needed
- **Better Scalability**: Horizontal scaling without database constraints
- **Managed Infrastructure**: Database maintenance handled by Supabase

## 3. Dockerfile Configuration

### 3.1 Updated Dockerfile

```dockerfile
# syntax=docker/dockerfile:1

# Base image with Node.js 20
FROM node:20-slim AS base

# Environment variables
ENV PLAYWRIGHT_SKIP_BROWSER_GC=1
ENV NODE_ENV=production

# Install global dependencies
RUN npm install -g dotenv-cli

# Create application user
RUN userdel -r node || true
RUN useradd -m -u 1000 user
USER user

# Set up user environment
ENV HOME=/home/user \
    PATH=/home/user/.local/bin:$PATH

WORKDIR /app

# Create .env.local placeholder
RUN touch /app/.env.local

# Install Playwright for web scraping
RUN npm i --no-package-lock --no-save playwright@1.52.0

# Switch to root for system dependencies
USER root

# Create directories and set permissions
RUN mkdir -p /data/models && \
    chown -R 1000:1000 /data/models

# Install system dependencies
RUN apt-get update && \
    apt-get install -y gnupg curl git cmake clang libgomp1 && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Install Playwright browsers
RUN npx playwright install --with-deps chromium

# Fix npm permissions
RUN chown -R 1000:1000 /home/user/.npm

# Switch back to application user
USER user

# Copy configuration files
COPY --chown=1000:1000 .env /app/.env
COPY --chown=1000:1000 entrypoint.sh /app/entrypoint.sh
COPY --chown=1000:1000 package*.json /app/

# Make entrypoint executable
RUN chmod +x /app/entrypoint.sh

# Builder stage
FROM node:20 AS builder

WORKDIR /app

# Copy package files
COPY --link package*.json ./

# Build arguments
ARG APP_BASE=""
ARG PUBLIC_APP_COLOR=blue
ARG SKIP_LLAMA_CPP_BUILD
ARG PUBLIC_COMMIT_SHA

# Environment variables for build
ENV BODY_SIZE_LIMIT=15728640
ENV SKIP_LLAMA_CPP_BUILD=$SKIP_LLAMA_CPP_BUILD
ENV PUBLIC_COMMIT_SHA=$PUBLIC_COMMIT_SHA

# Install dependencies with cache mount
RUN --mount=type=cache,target=/app/.npm \
    npm set cache /app/.npm && \
    npm ci --only=production

# Copy source code
COPY --link --chown=1000:1000 . .

# Build application
RUN git config --global --add safe.directory /app && \
    npm run build

# Final production image
FROM base AS final

# Build arguments for runtime
ARG APP_BASE=""
ARG PUBLIC_APP_COLOR=blue
ARG PUBLIC_COMMIT_SHA

# Runtime environment variables
ENV PUBLIC_COMMIT_SHA=$PUBLIC_COMMIT_SHA
ENV BODY_SIZE_LIMIT=15728640
ENV MODELS_STORAGE_PATH=/data/models
ENV NODE_ENV=production

# Copy built application and dependencies
COPY --from=builder --chown=1000:1000 /app/build /app/build
COPY --from=builder --chown=1000:1000 /app/node_modules /app/node_modules
COPY --from=builder --chown=1000:1000 /app/package.json /app/package.json

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3000/healthcheck || exit 1

# Expose port
EXPOSE 3000

# Start application
CMD ["/bin/bash", "-c", "/app/entrypoint.sh"]
```

### 3.2 Dockerfile Optimizations

- **Multi-stage build**: Separates build and runtime environments
- **Layer caching**: Optimizes rebuild times
- **Security**: Non-root user execution
- **Health checks**: Container health monitoring
- **Resource efficiency**: Minimal production image

## 4. Entrypoint Script

### 4.1 Updated entrypoint.sh

```bash
#!/bin/bash
set -e

# Configuration
ENV_LOCAL_PATH=/app/.env.local
LOG_LEVEL=${LOG_LEVEL:-info}

# Function to log messages
log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "Starting Chat UI with Supabase integration..."

# Handle environment variables
if test -z "${DOTENV_LOCAL}" ; then
    if ! test -f "${ENV_LOCAL_PATH}" ; then
        log "WARNING: DOTENV_LOCAL not found and .env.local not mounted. Using default configuration."
    else
        log "Using mounted .env.local file"
    fi
else
    log "Creating .env.local from DOTENV_LOCAL environment variable"
    cat <<< "$DOTENV_LOCAL" > ${ENV_LOCAL_PATH}
fi

# Validate required Supabase environment variables
if [ -z "$SUPABASE_URL" ] && [ -z "$PUBLIC_SUPABASE_URL" ]; then
    log "ERROR: SUPABASE_URL or PUBLIC_SUPABASE_URL must be set"
    exit 1
fi

if [ -z "$SUPABASE_ANON_KEY" ] && [ -z "$PUBLIC_SUPABASE_ANON_KEY" ]; then
    log "ERROR: SUPABASE_ANON_KEY or PUBLIC_SUPABASE_ANON_KEY must be set"
    exit 1
fi

# Set application version
export PUBLIC_VERSION=$(node -p "require('./package.json').version")
log "Starting Chat UI version: $PUBLIC_VERSION"

# Validate Supabase connection (optional)
if [ "$VALIDATE_SUPABASE_CONNECTION" = "true" ]; then
    log "Validating Supabase connection..."
    # Add connection validation logic here
fi

# Start the application
log "Starting SvelteKit application on port 3000..."
exec dotenv -e /app/.env -c -- node /app/build/index.js -- --host 0.0.0.0 --port 3000
```

## 5. Docker Compose Configuration

### 5.1 Development Environment

```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  chat-ui:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        PUBLIC_APP_COLOR: "blue"
        APP_BASE: ""
    ports:
      - "3000:3000"
    environment:
      # Supabase Configuration
      - SUPABASE_URL=${SUPABASE_URL}
      - SUPABASE_ANON_KEY=${SUPABASE_ANON_KEY}
      - SUPABASE_SERVICE_ROLE_KEY=${SUPABASE_SERVICE_ROLE_KEY}
      - PUBLIC_SUPABASE_URL=${SUPABASE_URL}
      - PUBLIC_SUPABASE_ANON_KEY=${SUPABASE_ANON_KEY}
      
      # AI Model APIs
      - HF_TOKEN=${HF_TOKEN}
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      
      # Application Configuration
      - PUBLIC_APP_NAME=ChatUI
      - PUBLIC_APP_ASSETS=chatui
      - LOG_LEVEL=debug
      - NODE_ENV=development
      
      # Optional: Validate connection on startup
      - VALIDATE_SUPABASE_CONNECTION=true
    
    volumes:
      # Mount models directory for local development
      - ./models:/data/models
      # Mount .env.local for local overrides
      - ./.env.local:/app/.env.local:ro
      # Mount source code for development (optional)
      - ./src:/app/src:ro
    
    restart: unless-stopped
    
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/healthcheck"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    
    # Resource limits for development
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: '1.0'
        reservations:
          memory: 512M
          cpus: '0.25'

# Optional: Add monitoring services
  watchtower:
    image: containrrr/watchtower
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    command: --interval 30 --cleanup
    restart: unless-stopped
```

### 5.2 Production Environment

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  chat-ui:
    image: chat-ui:latest
    ports:
      - "3000:3000"
    environment:
      # Supabase Configuration (use secrets in production)
      - SUPABASE_URL=${SUPABASE_URL}
      - SUPABASE_ANON_KEY=${SUPABASE_ANON_KEY}
      - SUPABASE_SERVICE_ROLE_KEY=${SUPABASE_SERVICE_ROLE_KEY}
      - PUBLIC_SUPABASE_URL=${SUPABASE_URL}
      - PUBLIC_SUPABASE_ANON_KEY=${SUPABASE_ANON_KEY}
      
      # Production settings
      - NODE_ENV=production
      - LOG_LEVEL=info
      - PUBLIC_APP_NAME=ChatUI
      - PUBLIC_APP_ASSETS=chatui
    
    volumes:
      - models_data:/data/models
    
    restart: always
    
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/healthcheck"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    
    # Production resource limits
    deploy:
      resources:
        limits:
          memory: 4G
          cpus: '2.0'
        reservations:
          memory: 1G
          cpus: '0.5'
      restart_policy:
        condition: any
        delay: 5s
        max_attempts: 3
        window: 120s
    
    # Security settings
    security_opt:
      - no-new-privileges:true
    read_only: true
    tmpfs:
      - /tmp
      - /var/tmp
    
    # Logging configuration
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # Reverse proxy (optional)
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - chat-ui
    restart: always

volumes:
  models_data:
    driver: local

networks:
  default:
    driver: bridge
```

## 6. Environment Variables

### 6.1 Required Variables

```bash
# .env.production

# Supabase Configuration
SUPABASE_URL=https://ybibjftiefxmnznkrojm.supabase.co
SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
SUPABASE_SERVICE_ROLE_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# Public Environment Variables (exposed to frontend)
PUBLIC_SUPABASE_URL=https://ybibjftiefxmnznkrojm.supabase.co
PUBLIC_SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# AI Model APIs
HF_TOKEN=hf_your_token_here
OPENAI_API_KEY=sk-your_openai_key_here
ANTHROPIC_API_KEY=sk-ant-your_anthropic_key_here

# Application Configuration
PUBLIC_APP_NAME=ChatUI
PUBLIC_APP_ASSETS=chatui
PUBLIC_APP_COLOR=blue

# Security
COOKIE_NAME=supabase-auth-token
COOKIE_SECURE=true
COOKIE_SAMESITE=strict

# Performance
BODY_SIZE_LIMIT=15728640
MODELS_STORAGE_PATH=/data/models

# Logging
LOG_LEVEL=info
NODE_ENV=production
```

### 6.2 Optional Variables

```bash
# Web Search APIs (optional)
YDC_API_KEY=your_ydc_api_key
SERPER_API_KEY=your_serper_api_key
SERPAPI_KEY=your_serpapi_key

# Feature Flags
ENABLE_ASSISTANTS=true
ENABLE_ASSISTANTS_RAG=false
COMMUNITY_TOOLS=false
ALLOW_IFRAME=true
ENABLE_DATA_EXPORT=true

# Rate Limiting
USAGE_LIMITS='{"conversations": 100, "messages": 1000, "assistants": 10}'

# Analytics (optional)
PUBLIC_GOOGLE_ANALYTICS_ID=G-XXXXXXXXXX
PUBLIC_PLAUSIBLE_SCRIPT_URL=/js/script.js

# Monitoring
METRICS_ENABLED=true
METRICS_PORT=5565
```

## 7. Build and Deployment Commands

### 7.1 Local Development

```bash
# Build the image
docker build -t chat-ui-dev .

# Run with docker-compose
docker-compose -f docker-compose.dev.yml up -d

# View logs
docker-compose -f docker-compose.dev.yml logs -f chat-ui

# Stop services
docker-compose -f docker-compose.dev.yml down
```

### 7.2 Production Deployment

```bash
# Build production image
docker build \
  --build-arg PUBLIC_APP_COLOR=blue \
  --build-arg PUBLIC_COMMIT_SHA=$(git rev-parse HEAD) \
  -t chat-ui:$(git rev-parse --short HEAD) \
  -t chat-ui:latest .

# Deploy with docker-compose
docker-compose -f docker-compose.prod.yml up -d

# Health check
docker-compose -f docker-compose.prod.yml ps
curl -f http://localhost:3000/healthcheck

# Update deployment
docker-compose -f docker-compose.prod.yml pull
docker-compose -f docker-compose.prod.yml up -d
```

### 7.3 Container Registry

```bash
# Tag for registry
docker tag chat-ui:latest your-registry.com/chat-ui:latest

# Push to registry
docker push your-registry.com/chat-ui:latest

# Pull and deploy
docker pull your-registry.com/chat-ui:latest
docker run -d \
  --name chat-ui \
  --env-file .env.production \
  -p 3000:3000 \
  your-registry.com/chat-ui:latest
```

## 8. Monitoring and Logging

### 8.1 Health Checks

```bash
# Container health status
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Application health endpoint
curl -f http://localhost:3000/healthcheck

# Detailed health information
curl -s http://localhost:3000/healthcheck | jq .
```

### 8.2 Log Management

```bash
# View application logs
docker logs chat-ui --follow --tail 100

# Filter logs by level
docker logs chat-ui 2>&1 | grep "ERROR"

# Export logs
docker logs chat-ui > chat-ui.log 2>&1
```

### 8.3 Performance Monitoring

```bash
# Container resource usage
docker stats chat-ui

# Memory usage details
docker exec chat-ui cat /proc/meminfo

# Process information
docker exec chat-ui ps aux
```

## 9. Security Considerations

### 9.1 Container Security

- **Non-root user**: Application runs as user ID 1000
- **Read-only filesystem**: Production containers use read-only root filesystem
- **No new privileges**: Security option prevents privilege escalation
- **Minimal attack surface**: Only necessary packages installed
- **Regular updates**: Base image and dependencies kept current

### 9.2 Environment Security

- **Secret management**: Use Docker secrets or external secret managers
- **Network isolation**: Containers run in isolated networks
- **TLS encryption**: HTTPS termination at reverse proxy
- **Environment variable protection**: Sensitive data not logged

### 9.3 Supabase Security

- **Row Level Security**: Database access controlled by RLS policies
- **JWT tokens**: Secure authentication with automatic token refresh
- **API key rotation**: Regular rotation of Supabase keys
- **Network restrictions**: Supabase project configured with allowed origins

## 10. Troubleshooting

### 10.1 Common Issues

**Container won't start:**
```bash
# Check container logs
docker logs chat-ui

# Verify environment variables
docker exec chat-ui env | grep SUPABASE

# Test Supabase connectivity
docker exec chat-ui curl -f $SUPABASE_URL/rest/v1/
```

**Application errors:**
```bash
# Check application health
curl -f http://localhost:3000/healthcheck

# Verify Supabase configuration
docker exec chat-ui node -e "console.log(process.env.PUBLIC_SUPABASE_URL)"

# Test database connection
docker exec chat-ui node -e "require('@supabase/supabase-js').createClient(process.env.SUPABASE_URL, process.env.SUPABASE_ANON_KEY).from('profiles').select('count')"
```

**Performance issues:**
```bash
# Monitor resource usage
docker stats chat-ui

# Check memory leaks
docker exec chat-ui node --expose-gc -e "global.gc(); console.log(process.memoryUsage())"

# Analyze slow queries
# Check Supabase dashboard for query performance
```

### 10.2 Recovery Procedures

**Container recovery:**
```bash
# Restart container
docker restart chat-ui

# Recreate container
docker-compose down && docker-compose up -d

# Rollback to previous version
docker run -d --name chat-ui-rollback your-registry.com/chat-ui:previous-tag
```

**Data recovery:**
- Supabase handles database backups automatically
- Point-in-time recovery available through Supabase dashboard
- Application state is stateless, no local data to recover

This Docker deployment configuration provides a robust, scalable, and secure foundation for running the Chat UI application with Supabase integration in containerized environments.