# Deployment Guide

> Production deployment guide for ThreatCopilot

---

## Table of Contents

1. [Deployment Options](#deployment-options)
2. [Docker Deployment](#docker-deployment)
3. [Kubernetes Deployment](#kubernetes-deployment)
4. [Cloud Deployment](#cloud-deployment)
5. [Security Hardening](#security-hardening)
6. [Monitoring & Maintenance](#monitoring--maintenance)

---

## Deployment Options

### Comparison

| Option | Complexity | Scalability | Cost | Best For |
|--------|-----------|-------------|------|----------|
| **Docker Compose** | Low | Low | $ | Development, small teams |
| **Kubernetes** | High | High | $$$ | Enterprise, high availability |
| **AWS ECS** | Medium | High | $$ | AWS-native deployments |
| **On-Premise** | Medium | Medium | $$ | Air-gapped environments |

---

## Docker Deployment

### Production Docker Compose

Create `docker-compose.prod.yml`:

```yaml
version: '3.8'

services:
  # PostgreSQL with replication
  postgres-primary:
    image: postgres:15-alpine
    container_name: threatcopilot-db-primary
    environment:
      POSTGRES_DB: threatcopilot
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_REPLICATION_MODE: master
      POSTGRES_REPLICATION_USER: replicator
      POSTGRES_REPLICATION_PASSWORD: ${REPLICATION_PASSWORD}
    volumes:
      - postgres_primary_data:/var/lib/postgresql/data
      - ./backups:/backups
    ports:
      - "5432:5432"
    networks:
      - threatcopilot-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  postgres-replica:
    image: postgres:15-alpine
    container_name: threatcopilot-db-replica
    environment:
      POSTGRES_REPLICATION_MODE: slave
      POSTGRES_MASTER_SERVICE: postgres-primary
      POSTGRES_REPLICATION_USER: replicator
      POSTGRES_REPLICATION_PASSWORD: ${REPLICATION_PASSWORD}
    volumes:
      - postgres_replica_data:/var/lib/postgresql/data
    networks:
      - threatcopilot-network
    restart: unless-stopped
    depends_on:
      - postgres-primary

  # Elasticsearch cluster
  elasticsearch-master:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.10.0
    container_name: threatcopilot-es-master
    environment:
      - node.name=es-master
      - cluster.name=threatcopilot-cluster
      - discovery.seed_hosts=elasticsearch-data1,elasticsearch-data2
      - cluster.initial_master_nodes=es-master
      - xpack.security.enabled=true
      - xpack.security.transport.ssl.enabled=true
      - ELASTIC_PASSWORD=${ES_PASSWORD}
      - "ES_JAVA_OPTS=-Xms4g -Xmx4g"
    volumes:
      - es_master_data:/usr/share/elasticsearch/data
      - ./config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
    ports:
      - "9200:9200"
    networks:
      - threatcopilot-network
    restart: unless-stopped

  elasticsearch-data1:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.10.0
    container_name: threatcopilot-es-data1
    environment:
      - node.name=es-data1
      - cluster.name=threatcopilot-cluster
      - discovery.seed_hosts=elasticsearch-master
      - cluster.initial_master_nodes=es-master
      - "ES_JAVA_OPTS=-Xms4g -Xmx4g"
    volumes:
      - es_data1:/usr/share/elasticsearch/data
    networks:
      - threatcopilot-network
    restart: unless-stopped

  # Redis for caching
  redis:
    image: redis:7-alpine
    container_name: threatcopilot-redis
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    networks:
      - threatcopilot-network
    restart: unless-stopped

  # Ollama AI service
  ollama:
    image: ollama/ollama:latest
    container_name: threatcopilot-ollama
    volumes:
      - ollama_data:/root/.ollama
    ports:
      - "11434:11434"
    networks:
      - threatcopilot-network
    restart: unless-stopped
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]

  # FastAPI backend (multiple instances)
  backend-1:
    build:
      context: .
      dockerfile: Dockerfile.prod
    container_name: threatcopilot-api-1
    environment:
      - DATABASE_URL=postgresql://${DB_USER}:${DB_PASSWORD}@postgres-primary:5432/threatcopilot
      - REDIS_URL=redis://:${REDIS_PASSWORD}@redis:6379/0
      - ELASTICSEARCH_HOST=elasticsearch-master
      - OLLAMA_HOST=http://ollama:11434
      - ENVIRONMENT=production
    volumes:
      - ./logs:/app/logs
    networks:
      - threatcopilot-network
    restart: unless-stopped
    depends_on:
      - postgres-primary
      - elasticsearch-master
      - redis
      - ollama

  backend-2:
    build:
      context: .
      dockerfile: Dockerfile.prod
    container_name: threatcopilot-api-2
    environment:
      - DATABASE_URL=postgresql://${DB_USER}:${DB_PASSWORD}@postgres-primary:5432/threatcopilot
      - REDIS_URL=redis://:${REDIS_PASSWORD}@redis:6379/0
      - ELASTICSEARCH_HOST=elasticsearch-master
      - OLLAMA_HOST=http://ollama:11434
      - ENVIRONMENT=production
    volumes:
      - ./logs:/app/logs
    networks:
      - threatcopilot-network
    restart: unless-stopped
    depends_on:
      - postgres-primary
      - elasticsearch-master
      - redis
      - ollama

  # Nginx load balancer
  nginx:
    image: nginx:alpine
    container_name: threatcopilot-nginx
    volumes:
      - ./config/nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    ports:
      - "80:80"
      - "443:443"
    networks:
      - threatcopilot-network
    restart: unless-stopped
    depends_on:
      - backend-1
      - backend-2

  # Prometheus monitoring
  prometheus:
    image: prom/prometheus:latest
    container_name: threatcopilot-prometheus
    volumes:
      - ./config/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - threatcopilot-network
    restart: unless-stopped

  # Grafana dashboards
  grafana:
    image: grafana/grafana:latest
    container_name: threatcopilot-grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
    volumes:
      - grafana_data:/var/lib/grafana
      - ./config/grafana/dashboards:/etc/grafana/provisioning/dashboards
    ports:
      - "3000:3000"
    networks:
      - threatcopilot-network
    restart: unless-stopped

volumes:
  postgres_primary_data:
  postgres_replica_data:
  es_master_data:
  es_data1:
  redis_data:
  ollama_data:
  prometheus_data:
  grafana_data:

networks:
  threatcopilot-network:
    driver: bridge
```

### Production Dockerfile

`Dockerfile.prod`:

```dockerfile
FROM python:3.11-slim

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Create app user
RUN useradd -m -u 1000 appuser

WORKDIR /app

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY --chown=appuser:appuser . .

# Switch to non-root user
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD python -c "import requests; requests.get('http://localhost:8000/health')"

# Run application
CMD ["gunicorn", "app.main:app", \
     "--workers", "4", \
     "--worker-class", "uvicorn.workers.UvicornWorker", \
     "--bind", "0.0.0.0:8000", \
     "--timeout", "120", \
     "--access-logfile", "-", \
     "--error-logfile", "-"]
```

### Nginx Configuration

`config/nginx.conf`:

```nginx
upstream backend {
    least_conn;
    server backend-1:8000 max_fails=3 fail_timeout=30s;
    server backend-2:8000 max_fails=3 fail_timeout=30s;
}

# Rate limiting
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=100r/m;

server {
    listen 80;
    server_name threatcopilot.yourdomain.com;
    
    # Redirect to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name threatcopilot.yourdomain.com;

    # SSL configuration
    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # API endpoints
    location /api/ {
        limit_req zone=api_limit burst=20 nodelay;
        
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # WebSocket
    location /ws/ {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }

    # Health check
    location /health {
        proxy_pass http://backend;
        access_log off;
    }
}
```

### Deploy to Production

```bash
# Set environment variables
export DB_USER=threatcopilot
export DB_PASSWORD=$(openssl rand -base64 32)
export REDIS_PASSWORD=$(openssl rand -base64 32)
export ES_PASSWORD=$(openssl rand -base64 32)

# Generate SSL certificates
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout ssl/key.pem -out ssl/cert.pem

# Deploy
docker-compose -f docker-compose.prod.yml up -d

# Initialize database
docker-compose -f docker-compose.prod.yml exec backend-1 \
  alembic upgrade head

# Pull Ollama model
docker-compose -f docker-compose.prod.yml exec ollama \
  ollama pull llama3

# Verify deployment
curl https://threatcopilot.yourdomain.com/health
```

---

## Kubernetes Deployment

### Kubernetes Manifests

`k8s/namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: threatcopilot
```

`k8s/configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: threatcopilot-config
  namespace: threatcopilot
data:
  ENVIRONMENT: "production"
  LOG_LEVEL: "INFO"
  ELASTICSEARCH_HOST: "elasticsearch-service"
  OLLAMA_HOST: "http://ollama-service:11434"
```

`k8s/secrets.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: threatcopilot-secrets
  namespace: threatcopilot
type: Opaque
stringData:
  DB_PASSWORD: "your-secure-password"
  REDIS_PASSWORD: "your-redis-password"
  ES_PASSWORD: "your-es-password"
  JWT_SECRET: "your-jwt-secret"
```

`k8s/postgres-statefulset.yaml`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: threatcopilot
spec:
  serviceName: postgres-service
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: threatcopilot
        - name: POSTGRES_USER
          value: threatcopilot
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: threatcopilot-secrets
              key: DB_PASSWORD
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Gi
```

`k8s/backend-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: threatcopilot
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: threatcopilot/backend:latest
        ports:
        - containerPort: 8000
        env:
        - name: DATABASE_URL
          value: "postgresql://threatcopilot:$(DB_PASSWORD)@postgres-service:5432/threatcopilot"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: threatcopilot-secrets
              key: DB_PASSWORD
        envFrom:
        - configMapRef:
            name: threatcopilot-config
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
```

`k8s/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: threatcopilot
spec:
  selector:
    app: backend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8000
  type: LoadBalancer
```

`k8s/ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: threatcopilot-ingress
  namespace: threatcopilot
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - threatcopilot.yourdomain.com
    secretName: threatcopilot-tls
  rules:
  - host: threatcopilot.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 80
```

### Deploy to Kubernetes

```bash
# Create namespace
kubectl apply -f k8s/namespace.yaml

# Create secrets
kubectl apply -f k8s/secrets.yaml

# Create configmap
kubectl apply -f k8s/configmap.yaml

# Deploy database
kubectl apply -f k8s/postgres-statefulset.yaml

# Deploy backend
kubectl apply -f k8s/backend-deployment.yaml

# Create services
kubectl apply -f k8s/service.yaml

# Create ingress
kubectl apply -f k8s/ingress.yaml

# Verify deployment
kubectl get pods -n threatcopilot
kubectl get services -n threatcopilot
```

### Helm Chart

`helm/threatcopilot/values.yaml`:

```yaml
replicaCount: 3

image:
  repository: threatcopilot/backend
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: LoadBalancer
  port: 80

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: threatcopilot.yourdomain.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: threatcopilot-tls
      hosts:
        - threatcopilot.yourdomain.com

resources:
  limits:
    cpu: 2000m
    memory: 4Gi
  requests:
    cpu: 1000m
    memory: 2Gi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

postgresql:
  enabled: true
  auth:
    database: threatcopilot
    username: threatcopilot
  primary:
    persistence:
      size: 100Gi

elasticsearch:
  enabled: true
  replicas: 3
  volumeClaimTemplate:
    resources:
      requests:
        storage: 200Gi

redis:
  enabled: true
  auth:
    enabled: true
```

Deploy with Helm:

```bash
# Add Helm repo
helm repo add threatcopilot https://charts.threatcopilot.ai

# Install
helm install threatcopilot threatcopilot/threatcopilot \
  --namespace threatcopilot \
  --create-namespace \
  --values values.yaml

# Upgrade
helm upgrade threatcopilot threatcopilot/threatcopilot \
  --namespace threatcopilot \
  --values values.yaml
```

---

## Cloud Deployment

### AWS ECS

`ecs-task-definition.json`:

```json
{
  "family": "threatcopilot-backend",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "2048",
  "memory": "4096",
  "containerDefinitions": [
    {
      "name": "backend",
      "image": "threatcopilot/backend:latest",
      "portMappings": [
        {
          "containerPort": 8000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "ENVIRONMENT",
          "value": "production"
        }
      ],
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:region:account:secret:db-password"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/threatcopilot",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "backend"
        }
      }
    }
  ]
}
```

Deploy to ECS:

```bash
# Create ECS cluster
aws ecs create-cluster --cluster-name threatcopilot-cluster

# Register task definition
aws ecs register-task-definition \
  --cli-input-json file://ecs-task-definition.json

# Create service
aws ecs create-service \
  --cluster threatcopilot-cluster \
  --service-name threatcopilot-backend \
  --task-definition threatcopilot-backend \
  --desired-count 3 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx],securityGroups=[sg-xxx],assignPublicIp=ENABLED}"
```

### Terraform

`main.tf`:

```hcl
provider "aws" {
  region = "us-east-1"
}

module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  
  name = "threatcopilot-vpc"
  cidr = "10.0.0.0/16"
  
  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  enable_nat_gateway = true
  enable_vpn_gateway = false
}

module "ecs" {
  source = "terraform-aws-modules/ecs/aws"
  
  cluster_name = "threatcopilot-cluster"
  
  fargate_capacity_providers = {
    FARGATE = {
      default_capacity_provider_strategy = {
        weight = 50
      }
    }
    FARGATE_SPOT = {
      default_capacity_provider_strategy = {
        weight = 50
      }
    }
  }
}

module "rds" {
  source = "terraform-aws-modules/rds/aws"
  
  identifier = "threatcopilot-db"
  
  engine            = "postgres"
  engine_version    = "15.3"
  instance_class    = "db.t3.large"
  allocated_storage = 100
  
  db_name  = "threatcopilot"
  username = "threatcopilot"
  password = var.db_password
  
  vpc_security_group_ids = [module.security_group.security_group_id]
  db_subnet_group_name   = module.vpc.database_subnet_group_name
  
  backup_retention_period = 7
  backup_window          = "03:00-06:00"
  maintenance_window     = "Mon:00:00-Mon:03:00"
}
```

---

## Security Hardening

### SSL/TLS Configuration

```bash
# Generate strong SSL certificate
openssl req -x509 -nodes -days 365 -newkey rsa:4096 \
  -keyout /etc/ssl/private/threatcopilot.key \
  -out /etc/ssl/certs/threatcopilot.crt

# Set proper permissions
chmod 600 /etc/ssl/private/threatcopilot.key
chmod 644 /etc/ssl/certs/threatcopilot.crt
```

### Firewall Rules

```bash
# Allow only necessary ports
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp   # SSH
ufw allow 80/tcp   # HTTP
ufw allow 443/tcp  # HTTPS
ufw enable
```

### Database Security

```sql
-- Create read-only user
CREATE USER threatcopilot_readonly WITH PASSWORD 'secure_password';
GRANT CONNECT ON DATABASE threatcopilot TO threatcopilot_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO threatcopilot_readonly;

-- Enable SSL
ALTER SYSTEM SET ssl = on;
ALTER SYSTEM SET ssl_cert_file = '/etc/ssl/certs/server.crt';
ALTER SYSTEM SET ssl_key_file = '/etc/ssl/private/server.key';
```

---

## Monitoring & Maintenance

### Backup Strategy

```bash
#!/bin/bash
# backup.sh

# PostgreSQL backup
docker-compose exec -T postgres pg_dump -U threatcopilot threatcopilot | \
  gzip > /backups/postgres_$(date +%Y%m%d_%H%M%S).sql.gz

# Elasticsearch snapshot
curl -X PUT "localhost:9200/_snapshot/backup_repo/snapshot_$(date +%Y%m%d)" \
  -H 'Content-Type: application/json' \
  -d '{"indices": "threat_logs_*"}'

# Rotate old backups (keep 30 days)
find /backups -name "*.sql.gz" -mtime +30 -delete
```

### Health Monitoring

```python
# monitoring/health_check.py

import requests
import time
from prometheus_client import start_http_server, Gauge

# Metrics
health_status = Gauge('threatcopilot_health', 'Health status')
response_time = Gauge('threatcopilot_response_time', 'Response time in ms')

def check_health():
    try:
        start = time.time()
        response = requests.get('http://localhost:8000/health', timeout=10)
        duration = (time.time() - start) * 1000
        
        if response.status_code == 200:
            health_status.set(1)
        else:
            health_status.set(0)
        
        response_time.set(duration)
    except Exception as e:
        health_status.set(0)
        print(f"Health check failed: {e}")

if __name__ == '__main__':
    start_http_server(8001)
    while True:
        check_health()
        time.sleep(30)
```

---

## Next Steps

- [Installation Guide](installation.md) - Set up ThreatCopilot
- [Architecture Deep Dive](architecture.md) - Understand system design
- [API Reference](api.md) - Explore API endpoints
- [Agent Configuration](agents.md) - Customize AI agents

