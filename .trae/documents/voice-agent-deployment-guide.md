# Voice Agent Deployment Guide

## 1. Overview

This guide provides comprehensive instructions for deploying the Chatterbox TTS voice agent integration in production environments, including Docker configuration, environment setup, monitoring, and troubleshooting.

## 2. Pre-Deployment Checklist

### 2.1 Requirements Verification

- [ ] RunPod account with active Chatterbox TTS endpoint
- [ ] Valid RunPod API key with appropriate permissions
- [ ] SvelteKit application with Supabase integration
- [ ] SSL certificate for HTTPS (required for Web Speech API)
- [ ] CDN configuration for audio file delivery
- [ ] Monitoring and logging infrastructure

### 2.2 Environment Configuration

```bash
# Production Environment Variables
# Core Configuration
NODE_ENV=production
PUBLIC_SUPABASE_URL=your_supabase_url
PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key
SUPABASE_SERVICE_ROLE_KEY=your_service_role_key

# Chatterbox TTS Configuration
CHATTERBOX_RUNPOD_ENDPOINT_ID=your_production_endpoint_id
CHATTERBOX_RUNPOD_API_KEY=your_production_api_key
CHATTERBOX_BASE_URL=https://api.runpod.ai/v2
CHATTERBOX_TIMEOUT_MS=45000
CHATTERBOX_MAX_RETRIES=3
CHATTERBOX_USE_ASYNC=true

# Rate Limiting
CHATTERBOX_RATE_LIMIT_RPM=120
CHATTERBOX_RATE_LIMIT_BURST=15

# Caching Configuration
CHATTERBOX_CACHE_ENABLED=true
CHATTERBOX_CACHE_TTL_HOURS=168
CHATTERBOX_CACHE_MAX_SIZE_MB=500

# Audio Configuration
AUDIO_MAX_DURATION_SECONDS=300
AUDIO_QUALITY=high
AUDIO_FORMAT=wav

# Security
VOICE_FEATURES_ENABLED=true
VOICE_REQUIRE_AUTH=true
VOICE_MAX_TEXT_LENGTH=2000

# Monitoring
MONITORING_ENABLED=true
LOG_LEVEL=info
METRICS_ENDPOINT=/api/metrics
```

## 3. Docker Configuration

### 3.1 Enhanced Dockerfile

```dockerfile
# Use the official Node.js image
FROM node:18-alpine AS base

# Install dependencies only when needed
FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app

# Copy package files
COPY package.json package-lock.json* ./
RUN npm ci --only=production

# Rebuild the source code only when needed
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Set build-time environment variables
ARG CHATTERBOX_RUNPOD_ENDPOINT_ID
ARG PUBLIC_SUPABASE_URL
ARG PUBLIC_SUPABASE_ANON_KEY

ENV CHATTERBOX_RUNPOD_ENDPOINT_ID=${CHATTERBOX_RUNPOD_ENDPOINT_ID}
ENV PUBLIC_SUPABASE_URL=${PUBLIC_SUPABASE_URL}
ENV PUBLIC_SUPABASE_ANON_KEY=${PUBLIC_SUPABASE_ANON_KEY}
ENV NODE_ENV=production

# Build the application
RUN npm run build

# Production image
FROM base AS runner
WORKDIR /app

# Create non-root user
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 sveltekit

# Copy built application
COPY --from=builder /app/build ./build
COPY --from=builder /app/package.json ./package.json
COPY --from=deps /app/node_modules ./node_modules

# Set ownership
RUN chown -R sveltekit:nodejs /app
USER sveltekit

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# Start the application
CMD ["node", "build"]
```

### 3.2 Docker Compose Configuration

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  chat-ui:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        - CHATTERBOX_RUNPOD_ENDPOINT_ID=${CHATTERBOX_RUNPOD_ENDPOINT_ID}
        - PUBLIC_SUPABASE_URL=${PUBLIC_SUPABASE_URL}
        - PUBLIC_SUPABASE_ANON_KEY=${PUBLIC_SUPABASE_ANON_KEY}
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - CHATTERBOX_RUNPOD_API_KEY=${CHATTERBOX_RUNPOD_API_KEY}
      - SUPABASE_SERVICE_ROLE_KEY=${SUPABASE_SERVICE_ROLE_KEY}
      - CHATTERBOX_USE_ASYNC=true
      - CHATTERBOX_RATE_LIMIT_RPM=120
      - CHATTERBOX_CACHE_ENABLED=true
      - VOICE_FEATURES_ENABLED=true
    volumes:
      - ./logs:/app/logs
      - audio_cache:/app/cache
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    networks:
      - chat_network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.chat-ui.rule=Host(`your-domain.com`)"
      - "traefik.http.routers.chat-ui.tls=true"
      - "traefik.http.routers.chat-ui.tls.certresolver=letsencrypt"

  # Redis for caching (optional)
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    restart: unless-stopped
    networks:
      - chat_network
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru

  # Nginx reverse proxy
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
      - nginx_logs:/var/log/nginx
    depends_on:
      - chat-ui
    restart: unless-stopped
    networks:
      - chat_network

volumes:
  redis_data:
  audio_cache:
  nginx_logs:

networks:
  chat_network:
    driver: bridge
```

### 3.3 Nginx Configuration

```nginx
# nginx.conf
events {
    worker_connections 1024;
}

http {
    upstream chat_ui {
        server chat-ui:3000;
    }
    
    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=voice:10m rate=5r/s;
    
    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css text/xml text/javascript application/javascript application/xml+rss application/json;
    
    # Security headers
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    
    server {
        listen 80;
        server_name your-domain.com;
        return 301 https://$server_name$request_uri;
    }
    
    server {
        listen 443 ssl http2;
        server_name your-domain.com;
        
        ssl_certificate /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384;
        ssl_prefer_server_ciphers off;
        
        # Main application
        location / {
            proxy_pass http://chat_ui;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_cache_bypass $http_upgrade;
            
            # Timeouts
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
        }
        
        # Voice API endpoints with special rate limiting
        location /api/voice {
            limit_req zone=voice burst=10 nodelay;
            
            proxy_pass http://chat_ui;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # Extended timeouts for TTS processing
            proxy_connect_timeout 120s;
            proxy_send_timeout 120s;
            proxy_read_timeout 120s;
        }
        
        # Static assets caching
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
            proxy_pass http://chat_ui;
        }
        
        # Health check endpoint
        location /health {
            access_log off;
            proxy_pass http://chat_ui;
        }
    }
}
```

## 4. Kubernetes Deployment

### 4.1 Kubernetes Manifests

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: chat-ui
  labels:
    name: chat-ui

---
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: chat-ui-config
  namespace: chat-ui
data:
  NODE_ENV: "production"
  CHATTERBOX_BASE_URL: "https://api.runpod.ai/v2"
  CHATTERBOX_USE_ASYNC: "true"
  CHATTERBOX_RATE_LIMIT_RPM: "120"
  CHATTERBOX_CACHE_ENABLED: "true"
  VOICE_FEATURES_ENABLED: "true"
  LOG_LEVEL: "info"

---
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: chat-ui-secrets
  namespace: chat-ui
type: Opaque
stringData:
  CHATTERBOX_RUNPOD_API_KEY: "your-api-key-here"
  CHATTERBOX_RUNPOD_ENDPOINT_ID: "your-endpoint-id-here"
  SUPABASE_SERVICE_ROLE_KEY: "your-supabase-key-here"

---
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chat-ui
  namespace: chat-ui
  labels:
    app: chat-ui
spec:
  replicas: 3
  selector:
    matchLabels:
      app: chat-ui
  template:
    metadata:
      labels:
        app: chat-ui
    spec:
      containers:
      - name: chat-ui
        image: your-registry/chat-ui:latest
        ports:
        - containerPort: 3000
        envFrom:
        - configMapRef:
            name: chat-ui-config
        - secretRef:
            name: chat-ui-secrets
        env:
        - name: PUBLIC_SUPABASE_URL
          value: "your-supabase-url"
        - name: PUBLIC_SUPABASE_ANON_KEY
          value: "your-supabase-anon-key"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
        volumeMounts:
        - name: audio-cache
          mountPath: /app/cache
      volumes:
      - name: audio-cache
        emptyDir:
          sizeLimit: 1Gi

---
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: chat-ui-service
  namespace: chat-ui
spec:
  selector:
    app: chat-ui
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000
  type: ClusterIP

---
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: chat-ui-ingress
  namespace: chat-ui
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/rate-limit: "10"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
spec:
  tls:
  - hosts:
    - your-domain.com
    secretName: chat-ui-tls
  rules:
  - host: your-domain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: chat-ui-service
            port:
              number: 80

---
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: chat-ui-hpa
  namespace: chat-ui
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: chat-ui
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

## 5. Health Checks and Monitoring

### 5.1 Health Check Endpoint

```typescript
// src/routes/health/+server.ts
import { json } from '@sveltejs/kit';
import { ttsService } from '$lib/services/ChatterboxTTSService';
import { runPodConfig } from '$lib/config/runpod';

export async function GET() {
  const health = {
    status: 'healthy',
    timestamp: new Date().toISOString(),
    version: process.env.npm_package_version || '1.0.0',
    services: {
      database: 'unknown',
      tts: 'unknown',
      cache: 'unknown'
    },
    metrics: {
      uptime: process.uptime(),
      memory: process.memoryUsage(),
      cpu: process.cpuUsage()
    }
  };
  
  try {
    // Check TTS service
    const ttsHealthy = await ttsService.testConnection();
    health.services.tts = ttsHealthy ? 'healthy' : 'unhealthy';
    
    // Check configuration
    const configValid = runPodConfig.validateConfig();
    if (!configValid) {
      health.services.tts = 'misconfigured';
    }
    
    // Check Supabase connection (if applicable)
    // Add your Supabase health check here
    
    // Determine overall status
    const allHealthy = Object.values(health.services).every(status => status === 'healthy');
    health.status = allHealthy ? 'healthy' : 'degraded';
    
    return json(health, {
      status: allHealthy ? 200 : 503
    });
    
  } catch (error) {
    health.status = 'unhealthy';
    health.error = error.message;
    
    return json(health, { status: 503 });
  }
}
```

### 5.2 Metrics Endpoint

```typescript
// src/routes/api/metrics/+server.ts
import { json } from '@sveltejs/kit';
import { ttsService } from '$lib/services/ChatterboxTTSService';
import { ttsMonitoring } from '$lib/services/TTSMonitoring';

export async function GET() {
  try {
    const metrics = {
      timestamp: new Date().toISOString(),
      tts: ttsMonitoring.getMetrics(),
      service: ttsService.getStats(),
      system: {
        uptime: process.uptime(),
        memory: process.memoryUsage(),
        cpu: process.cpuUsage()
      }
    };
    
    return json(metrics);
  } catch (error) {
    return json({ error: error.message }, { status: 500 });
  }
}
```

## 6. Logging Configuration

### 6.1 Structured Logging

```typescript
// src/lib/utils/logger.ts
import { browser } from '$app/environment';

export enum LogLevel {
  ERROR = 0,
  WARN = 1,
  INFO = 2,
  DEBUG = 3
}

export interface LogEntry {
  timestamp: string;
  level: string;
  message: string;
  service: string;
  metadata?: any;
  traceId?: string;
}

class Logger {
  private level: LogLevel;
  private service: string;
  
  constructor(service: string = 'chat-ui') {
    this.service = service;
    this.level = this.getLogLevel();
  }
  
  private getLogLevel(): LogLevel {
    const level = (browser ? import.meta.env.VITE_LOG_LEVEL : process.env.LOG_LEVEL) || 'info';
    
    switch (level.toLowerCase()) {
      case 'error': return LogLevel.ERROR;
      case 'warn': return LogLevel.WARN;
      case 'info': return LogLevel.INFO;
      case 'debug': return LogLevel.DEBUG;
      default: return LogLevel.INFO;
    }
  }
  
  private log(level: LogLevel, message: string, metadata?: any, traceId?: string): void {
    if (level > this.level) return;
    
    const entry: LogEntry = {
      timestamp: new Date().toISOString(),
      level: LogLevel[level],
      message,
      service: this.service,
      metadata,
      traceId
    };
    
    if (browser) {
      // Browser logging
      const method = level === LogLevel.ERROR ? 'error' : 
                    level === LogLevel.WARN ? 'warn' : 'log';
      console[method](`[${entry.timestamp}] ${entry.level}: ${entry.message}`, metadata);
    } else {
      // Server logging (structured JSON)
      console.log(JSON.stringify(entry));
    }
  }
  
  error(message: string, metadata?: any, traceId?: string): void {
    this.log(LogLevel.ERROR, message, metadata, traceId);
  }
  
  warn(message: string, metadata?: any, traceId?: string): void {
    this.log(LogLevel.WARN, message, metadata, traceId);
  }
  
  info(message: string, metadata?: any, traceId?: string): void {
    this.log(LogLevel.INFO, message, metadata, traceId);
  }
  
  debug(message: string, metadata?: any, traceId?: string): void {
    this.log(LogLevel.DEBUG, message, metadata, traceId);
  }
}

export const logger = new Logger();
export const ttsLogger = new Logger('tts-service');
export const voiceLogger = new Logger('voice-agent');
```

## 7. Performance Optimization

### 7.1 CDN Configuration

```typescript
// src/lib/config/cdn.ts
export interface CDNConfig {
  enabled: boolean;
  baseUrl: string;
  audioPath: string;
  cacheTTL: number;
}

export const cdnConfig: CDNConfig = {
  enabled: import.meta.env.VITE_CDN_ENABLED === 'true',
  baseUrl: import.meta.env.VITE_CDN_BASE_URL || '',
  audioPath: '/audio',
  cacheTTL: 86400 // 24 hours
};

export function getCDNUrl(audioUrl: string): string {
  if (!cdnConfig.enabled || !cdnConfig.baseUrl) {
    return audioUrl;
  }
  
  // Extract filename from RunPod URL
  const filename = audioUrl.split('/').pop();
  return `${cdnConfig.baseUrl}${cdnConfig.audioPath}/${filename}`;
}
```

### 7.2 Service Worker for Caching

```javascript
// src/service-worker.js
const CACHE_NAME = 'voice-agent-v1';
const AUDIO_CACHE = 'voice-audio-v1';

// Cache audio files
self.addEventListener('fetch', event => {
  const url = new URL(event.request.url);
  
  // Cache audio files
  if (url.pathname.includes('/audio/') || url.pathname.endsWith('.wav') || url.pathname.endsWith('.mp3')) {
    event.respondWith(
      caches.open(AUDIO_CACHE).then(cache => {
        return cache.match(event.request).then(response => {
          if (response) {
            return response;
          }
          
          return fetch(event.request).then(fetchResponse => {
            // Only cache successful responses
            if (fetchResponse.status === 200) {
              cache.put(event.request, fetchResponse.clone());
            }
            return fetchResponse;
          });
        });
      })
    );
  }
  
  // Cache API responses
  if (url.pathname.startsWith('/api/voice/')) {
    event.respondWith(
      caches.open(CACHE_NAME).then(cache => {
        return cache.match(event.request).then(response => {
          if (response) {
            // Serve from cache, but also fetch in background
            fetch(event.request).then(fetchResponse => {
              if (fetchResponse.status === 200) {
                cache.put(event.request, fetchResponse.clone());
              }
            });
            return response;
          }
          
          return fetch(event.request).then(fetchResponse => {
            if (fetchResponse.status === 200) {
              cache.put(event.request, fetchResponse.clone());
            }
            return fetchResponse;
          });
        });
      })
    );
  }
});

// Clean up old caches
self.addEventListener('activate', event => {
  event.waitUntil(
    caches.keys().then(cacheNames => {
      return Promise.all(
        cacheNames.map(cacheName => {
          if (cacheName !== CACHE_NAME && cacheName !== AUDIO_CACHE) {
            return caches.delete(cacheName);
          }
        })
      );
    })
  );
});
```

## 8. Security Configuration

### 8.1 Content Security Policy

```typescript
// src/app.html
<meta http-equiv="Content-Security-Policy" content="
  default-src 'self';
  script-src 'self' 'unsafe-inline' 'unsafe-eval';
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  font-src 'self' data:;
  connect-src 'self' https://api.runpod.ai https://*.supabase.co wss://*.supabase.co;
  media-src 'self' https: data: blob:;
  worker-src 'self' blob:;
  child-src 'self' blob:;
  frame-ancestors 'none';
  base-uri 'self';
  form-action 'self';
">
```

### 8.2 API Rate Limiting

```typescript
// src/lib/middleware/rateLimiter.ts
import type { RequestEvent } from '@sveltejs/kit';

interface RateLimitStore {
  [key: string]: {
    count: number;
    resetTime: number;
  };
}

const store: RateLimitStore = {};

export function rateLimit(options: {
  windowMs: number;
  maxRequests: number;
  keyGenerator?: (event: RequestEvent) => string;
}) {
  return async (event: RequestEvent) => {
    const key = options.keyGenerator ? 
      options.keyGenerator(event) : 
      event.getClientAddress();
    
    const now = Date.now();
    const windowStart = now - options.windowMs;
    
    // Clean up old entries
    Object.keys(store).forEach(k => {
      if (store[k].resetTime < windowStart) {
        delete store[k];
      }
    });
    
    // Check current request
    if (!store[key]) {
      store[key] = { count: 1, resetTime: now + options.windowMs };
      return;
    }
    
    if (store[key].count >= options.maxRequests) {
      throw new Error('Rate limit exceeded');
    }
    
    store[key].count++;
  };
}
```

## 9. Troubleshooting Guide

### 9.1 Common Issues and Solutions

| Issue | Symptoms | Solution |
|-------|----------|----------|
| TTS requests failing | 401/403 errors | Check API key and endpoint ID |
| Audio not playing | Silent playback | Verify HTTPS and audio format support |
| High latency | Slow voice responses | Enable async mode, check network |
| Memory leaks | Increasing memory usage | Implement proper cleanup in components |
| Rate limiting | 429 errors | Adjust rate limits or implement queuing |
| Cache issues | Stale audio responses | Clear cache or adjust TTL |

### 9.2 Debug Commands

```bash
# Check container logs
docker logs chat-ui-container

# Monitor resource usage
docker stats chat-ui-container

# Test TTS endpoint
curl -X POST "https://api.runpod.ai/v2/YOUR_ENDPOINT/health" \
  -H "Authorization: Bearer YOUR_API_KEY"

# Check application health
curl https://your-domain.com/health

# View metrics
curl https://your-domain.com/api/metrics
```

### 9.3 Performance Monitoring

```typescript
// src/lib/monitoring/performance.ts
export class PerformanceMonitor {
  private static instance: PerformanceMonitor;
  private metrics: Map<string, number[]> = new Map();
  
  static getInstance(): PerformanceMonitor {
    if (!this.instance) {
      this.instance = new PerformanceMonitor();
    }
    return this.instance;
  }
  
  startTimer(operation: string): () => void {
    const start = performance.now();
    
    return () => {
      const duration = performance.now() - start;
      this.recordMetric(operation, duration);
    };
  }
  
  recordMetric(operation: string, value: number): void {
    if (!this.metrics.has(operation)) {
      this.metrics.set(operation, []);
    }
    
    const values = this.metrics.get(operation)!;
    values.push(value);
    
    // Keep only last 100 measurements
    if (values.length > 100) {
      values.shift();
    }
  }
  
  getMetrics(): Record<string, {
    avg: number;
    min: number;
    max: number;
    count: number;
  }> {
    const result: Record<string, any> = {};
    
    for (const [operation, values] of this.metrics) {
      if (values.length > 0) {
        result[operation] = {
          avg: values.reduce((sum, val) => sum + val, 0) / values.length,
          min: Math.min(...values),
          max: Math.max(...values),
          count: values.length
        };
      }
    }
    
    return result;
  }
}

export const performanceMonitor = PerformanceMonitor.getInstance();
```

## 10. Deployment Scripts

### 10.1 Deployment Script

```bash
#!/bin/bash
# deploy.sh

set -e

echo "Starting deployment..."

# Build and tag image
echo "Building Docker image..."
docker build -t chat-ui:latest \
  --build-arg CHATTERBOX_RUNPOD_ENDPOINT_ID="$CHATTERBOX_RUNPOD_ENDPOINT_ID" \
  --build-arg PUBLIC_SUPABASE_URL="$PUBLIC_SUPABASE_URL" \
  --build-arg PUBLIC_SUPABASE_ANON_KEY="$PUBLIC_SUPABASE_ANON_KEY" \
  .

# Tag for registry
docker tag chat-ui:latest your-registry/chat-ui:latest
docker tag chat-ui:latest your-registry/chat-ui:$(git rev-parse --short HEAD)

# Push to registry
echo "Pushing to registry..."
docker push your-registry/chat-ui:latest
docker push your-registry/chat-ui:$(git rev-parse --short HEAD)

# Deploy to production
echo "Deploying to production..."
if [ "$DEPLOYMENT_TYPE" = "kubernetes" ]; then
  kubectl apply -f k8s/
  kubectl rollout restart deployment/chat-ui -n chat-ui
  kubectl rollout status deployment/chat-ui -n chat-ui
else
  docker-compose -f docker-compose.prod.yml up -d
fi

echo "Deployment completed successfully!"
```

### 10.2 Health Check Script

```bash
#!/bin/bash
# health-check.sh

HEALTH_URL="https://your-domain.com/health"
MAX_RETRIES=5
RETRY_DELAY=10

for i in $(seq 1 $MAX_RETRIES); do
  echo "Health check attempt $i/$MAX_RETRIES..."
  
  if curl -f -s "$HEALTH_URL" > /dev/null; then
    echo "✅ Application is healthy!"
    exit 0
  fi
  
  if [ $i -lt $MAX_RETRIES ]; then
    echo "❌ Health check failed, retrying in ${RETRY_DELAY}s..."
    sleep $RETRY_DELAY
  fi
done

echo "❌ Application failed health checks after $MAX_RETRIES attempts"
exit 1
```

This comprehensive deployment guide provides everything needed to successfully deploy the voice agent integration in production environments with proper monitoring, security, and performance optimization.