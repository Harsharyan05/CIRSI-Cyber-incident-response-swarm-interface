# Installation Guide

> Complete guide to installing and setting up ThreatCopilot on your infrastructure

---

## Table of Contents

1. [System Requirements](#system-requirements)
2. [Prerequisites](#prerequisites)
3. [Installation Methods](#installation-methods)
4. [Configuration](#configuration)
5. [Verification](#verification)
6. [Troubleshooting](#troubleshooting)

---

## System Requirements

### Minimum Requirements

| Component | Specification |
|-----------|--------------|
| **CPU** | 4 cores (x86_64) |
| **RAM** | 8GB |
| **Storage** | 50GB available space |
| **OS** | Linux (Ubuntu 20.04+), macOS (11+), Windows 10/11 with WSL2 |
| **Network** | 10Mbps (for initial setup) |

### Recommended Requirements

| Component | Specification |
|-----------|--------------|
| **CPU** | 8+ cores (x86_64) |
| **RAM** | 16GB+ |
| **Storage** | 100GB SSD |
| **OS** | Ubuntu 22.04 LTS or RHEL 8+ |
| **Network** | 100Mbps+ |

### Port Requirements

```
8000  - FastAPI Backend
5432  - PostgreSQL Database
9200  - Elasticsearch
9300  - Elasticsearch Cluster
11434 - Ollama AI Service
3000  - Frontend UI (Development)
```

---

## Prerequisites

### 1. Docker Installation

#### Linux (Ubuntu/Debian)
```bash
# Update package index
sudo apt-get update

# Install dependencies
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# Add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Set up stable repository
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io

# Verify installation
docker --version
```

#### macOS
```bash
# Install via Homebrew
brew install --cask docker

# Or download Docker Desktop from:
# https://www.docker.com/products/docker-desktop

# Verify installation
docker --version
```

#### Windows (WSL2)
```powershell
# Install WSL2
wsl --install

# Download and install Docker Desktop for Windows
# https://www.docker.com/products/docker-desktop

# Verify installation
docker --version
```

### 2. Docker Compose Installation

```bash
# Linux
sudo curl -L "https://github.com/docker/compose/releases/download/v2.20.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Verify installation
docker-compose --version
```

### 3. Git Installation

```bash
# Ubuntu/Debian
sudo apt-get install git

# macOS
brew install git

# Verify installation
git --version
```

---

## Installation Methods

### Method 1: Docker Compose (Recommended)

#### Step 1: Clone Repository
```bash
# Clone the repository
git clone https://github.com/yourusername/threatcopilot.git
cd threatcopilot
```

#### Step 2: Environment Configuration
```bash
# Copy environment template
cp .env.example .env

# Edit configuration (see Configuration section)
nano .env
```

#### Step 3: Deploy Services
```bash
# Pull and start all services
docker-compose up -d

# View startup logs
docker-compose logs -f
```

#### Step 4: Initialize Database
```bash
# Run database migrations
docker-compose exec backend python -m alembic upgrade head

# Seed initial data
docker-compose exec backend python scripts/seed_data.py
```

#### Step 5: Verify Installation
```bash
# Check all services are running
docker-compose ps

# Expected output:
# NAME                STATUS              PORTS
# threatcopilot-api   Up 2 minutes        0.0.0.0:8000->8000/tcp
# threatcopilot-db    Up 2 minutes        0.0.0.0:5432->5432/tcp
# threatcopilot-es    Up 2 minutes        0.0.0.0:9200->9200/tcp
# threatcopilot-ollama Up 2 minutes       0.0.0.0:11434->11434/tcp
```

### Method 2: Manual Installation

#### Step 1: Install Python Dependencies
```bash
# Create virtual environment
python3 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

#### Step 2: Install Ollama
```bash
# Linux
curl -fsSL https://ollama.ai/install.sh | sh

# macOS
brew install ollama

# Pull Llama3 model
ollama pull llama3
```

#### Step 3: Setup PostgreSQL
```bash
# Install PostgreSQL
sudo apt-get install postgresql postgresql-contrib

# Create database
sudo -u postgres psql
CREATE DATABASE threatcopilot;
CREATE USER threatcopilot_user WITH PASSWORD 'your_password';
GRANT ALL PRIVILEGES ON DATABASE threatcopilot TO threatcopilot_user;
\q
```

#### Step 4: Setup Elasticsearch
```bash
# Download and install Elasticsearch
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.10.0-linux-x86_64.tar.gz
tar -xzf elasticsearch-8.10.0-linux-x86_64.tar.gz
cd elasticsearch-8.10.0/

# Start Elasticsearch
./bin/elasticsearch -d
```

#### Step 5: Start Backend
```bash
# Run migrations
alembic upgrade head

# Start FastAPI server
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

---

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
# Application Settings
APP_NAME=ThreatCopilot
APP_VERSION=1.0.0
ENVIRONMENT=production
DEBUG=false

# API Configuration
API_HOST=0.0.0.0
API_PORT=8000
API_WORKERS=4

# Database Configuration
DATABASE_URL=postgresql://threatcopilot_user:your_password@localhost:5432/threatcopilot
DB_POOL_SIZE=20
DB_MAX_OVERFLOW=10

# Elasticsearch Configuration
ELASTICSEARCH_HOST=localhost
ELASTICSEARCH_PORT=9200
ELASTICSEARCH_INDEX=threat_logs

# Ollama Configuration
OLLAMA_HOST=http://localhost:11434
OLLAMA_MODEL=llama3
OLLAMA_TIMEOUT=120

# AI Agent Configuration
AGENT_TIMEOUT=30
AGENT_MAX_RETRIES=3
DETECTION_THRESHOLD=0.85

# ML Model Configuration
ML_MODEL_PATH=./models/anomaly_detector.pkl
ML_CONFIDENCE_THRESHOLD=0.92

# Security Settings
SECRET_KEY=your-secret-key-here-change-in-production
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30

# Logging Configuration
LOG_LEVEL=INFO
LOG_FILE=./logs/threatcopilot.log

# Phone Notification (Optional)
TWILIO_ACCOUNT_SID=your_twilio_sid
TWILIO_AUTH_TOKEN=your_twilio_token
TWILIO_PHONE_NUMBER=+1234567890
ALERT_PHONE_NUMBER=+1234567890

# Email Notification (Optional)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
ALERT_EMAIL=security@yourcompany.com
```

### Docker Compose Configuration

Edit `docker-compose.yml` for custom settings:

```yaml
version: '3.8'

services:
  # PostgreSQL Database
  postgres:
    image: postgres:15-alpine
    container_name: threatcopilot-db
    environment:
      POSTGRES_DB: threatcopilot
      POSTGRES_USER: threatcopilot_user
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U threatcopilot_user"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Elasticsearch
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.10.0
    container_name: threatcopilot-es
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms2g -Xmx2g"
    volumes:
      - es_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
      - "9300:9300"
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_cluster/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5

  # Ollama AI Service
  ollama:
    image: ollama/ollama:latest
    container_name: threatcopilot-ollama
    volumes:
      - ollama_data:/root/.ollama
    ports:
      - "11434:11434"
    command: serve

  # FastAPI Backend
  backend:
    build: .
    container_name: threatcopilot-api
    depends_on:
      - postgres
      - elasticsearch
      - ollama
    environment:
      - DATABASE_URL=postgresql://threatcopilot_user:${DB_PASSWORD}@postgres:5432/threatcopilot
      - ELASTICSEARCH_HOST=elasticsearch
      - OLLAMA_HOST=http://ollama:11434
    volumes:
      - ./app:/app
      - ./logs:/logs
    ports:
      - "8000:8000"
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000

volumes:
  postgres_data:
  es_data:
  ollama_data:
```

---

## Verification

### 1. Health Check Endpoints

```bash
# Check API health
curl http://localhost:8000/health

# Expected response:
{
  "status": "healthy",
  "version": "1.0.0",
  "services": {
    "database": "connected",
    "elasticsearch": "connected",
    "ollama": "connected"
  }
}
```

### 2. Test AI Agent

```bash
# Test threat detection
curl -X POST http://localhost:8000/api/v1/detect \
  -H "Content-Type: application/json" \
  -d '{
    "log_entry": "Failed login attempt from 192.168.1.100",
    "timestamp": "2024-01-15T10:30:00Z"
  }'

# Expected response:
{
  "threat_detected": true,
  "confidence": 0.92,
  "threat_type": "brute_force_attack",
  "recommended_actions": ["quarantine_ip", "alert_admin"]
}
```

### 3. Access Web Dashboard

```bash
# Open in browser
http://localhost:8000/dashboard

# Or scan QR code for mobile access
curl http://localhost:8000/api/qr-code
```

### 4. View Logs

```bash
# Application logs
docker-compose logs -f backend

# Database logs
docker-compose logs -f postgres

# Elasticsearch logs
docker-compose logs -f elasticsearch

# All services
docker-compose logs -f
```

---

## Troubleshooting

### Issue 1: Port Already in Use

```bash
# Find process using port 8000
sudo lsof -i :8000

# Kill the process
sudo kill -9 <PID>

# Or change port in docker-compose.yml
ports:
  - "8080:8000"  # Use 8080 instead
```

### Issue 2: Ollama Model Not Found

```bash
# Enter Ollama container
docker-compose exec ollama bash

# Pull Llama3 model
ollama pull llama3

# Verify model
ollama list
```

### Issue 3: Elasticsearch Memory Issues

```bash
# Increase Docker memory limit (Docker Desktop)
# Settings > Resources > Memory > 8GB+

# Or reduce ES memory in docker-compose.yml
environment:
  - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
```

### Issue 4: Database Connection Failed

```bash
# Check PostgreSQL is running
docker-compose ps postgres

# Check logs
docker-compose logs postgres

# Reset database
docker-compose down -v
docker-compose up -d
```

### Issue 5: Slow Performance

```bash
# Check resource usage
docker stats

# Optimize Docker resources
# - Increase CPU cores
# - Increase RAM allocation
# - Use SSD storage

# Enable production optimizations
ENVIRONMENT=production
DEBUG=false
```

### Common Error Messages

| Error | Solution |
|-------|----------|
| `Connection refused` | Check service is running: `docker-compose ps` |
| `Out of memory` | Increase Docker memory allocation |
| `Port already allocated` | Change port or kill conflicting process |
| `Model not found` | Pull Ollama model: `ollama pull llama3` |
| `Database migration failed` | Reset DB: `docker-compose down -v && docker-compose up -d` |

---

## Post-Installation Steps

### 1. Create Admin User

```bash
# Create admin account
docker-compose exec backend python scripts/create_admin.py \
  --username admin \
  --email admin@yourcompany.com \
  --password SecurePassword123!
```

### 2. Configure Notifications

```bash
# Test phone notification
curl -X POST http://localhost:8000/api/v1/test-notification \
  -H "Content-Type: application/json" \
  -d '{"type": "phone", "message": "Test alert"}'

# Test email notification
curl -X POST http://localhost:8000/api/v1/test-notification \
  -H "Content-Type: application/json" \
  -d '{"type": "email", "message": "Test alert"}'
```

### 3. Import Threat Intelligence

```bash
# Import threat feeds
docker-compose exec backend python scripts/import_threat_intel.py \
  --source mitre_attack \
  --update
```

### 4. Run Initial Scan

```bash
# Scan existing logs
curl -X POST http://localhost:8000/api/v1/scan \
  -H "Content-Type: application/json" \
  -d '{"scan_type": "full", "time_range": "24h"}'
```

---

## Updating ThreatCopilot

```bash
# Pull latest changes
git pull origin main

# Rebuild containers
docker-compose down
docker-compose build --no-cache
docker-compose up -d

# Run migrations
docker-compose exec backend alembic upgrade head
```

---

## Uninstallation

```bash
# Stop all services
docker-compose down

# Remove all data (WARNING: This deletes all data)
docker-compose down -v

# Remove Docker images
docker rmi $(docker images 'threatcopilot*' -q)

# Remove project directory
cd ..
rm -rf threatcopilot
```

---

## Next Steps

- [Architecture Deep Dive](architecture.md) - Understand system design
- [API Reference](api.md) - Explore API endpoints
- [Agent Configuration](agents.md) - Customize AI agents
- [Deployment Guide](deployment.md) - Deploy to production

---

## Support

- **Issues**: [GitHub Issues](https://github.com/yourusername/threatcopilot/issues)
- **Documentation**: [Full Docs](https://docs.threatcopilot.ai)
- **Email**: support@threatcopilot.ai

