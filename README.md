# CIRSI-Cyber-incident-report-system-interface-
# ThreatCopilot Documentation

Welcome to the comprehensive documentation for ThreatCopilot - the AI-powered autonomous Security Operations Center.

---

## 📚 Documentation Structure

### [Installation Guide](installation.md)
Complete setup instructions for ThreatCopilot on various platforms.

**Contents:**
- System requirements (minimum & recommended)
- Prerequisites (Docker, Docker Compose, Git)
- Installation methods (Docker Compose & Manual)
- Environment configuration
- Verification steps
- Troubleshooting common issues
- Post-installation setup

**Start here if:** You're setting up ThreatCopilot for the first time.

---

### [Architecture Deep Dive](architecture.md)
Technical architecture and system design documentation.

**Contents:**
- 5-layer system architecture
- Component breakdown (API Gateway, AI Orchestration, Intelligence, Data layers)
- Data flow and 18-second response timeline
- AI agent system architecture
- Security architecture (defense in depth)
- Performance optimization strategies
- Monitoring and observability

**Start here if:** You want to understand how ThreatCopilot works internally.

---

### [API Reference](api.md)
Complete REST API documentation with examples.

**Contents:**
- Authentication (JWT tokens)
- All API endpoints with request/response examples
- WebSocket API for real-time updates
- Error handling and status codes
- Rate limiting policies
- Code examples (Python, JavaScript, cURL)

**Start here if:** You're integrating ThreatCopilot with other systems.

---

### [Agent Configuration](agents.md)
Guide to configuring and customizing AI agents.

**Contents:**
- Detector Agent configuration (ML models, detection rules)
- Analyzer Agent configuration (threat classification, impact calculation)
- Executor Agent configuration (playbooks, custom actions)
- Creating custom agents
- Agent tuning and optimization
- A/B testing strategies

**Start here if:** You want to customize threat detection and response behavior.

---

### [Deployment Guide](deployment.md)
Production deployment strategies and best practices.

**Contents:**
- Deployment options comparison
- Docker production deployment
- Kubernetes deployment with Helm
- Cloud deployment (AWS ECS, Terraform)
- Security hardening
- Monitoring and maintenance
- Backup strategies

**Start here if:** You're deploying ThreatCopilot to production.

---

## 🚀 Quick Start Path

### For Developers
1. [Installation Guide](installation.md) → Set up local environment
2. [API Reference](api.md) → Explore API endpoints
3. [Architecture Deep Dive](architecture.md) → Understand the system

### For Security Engineers
1. [Installation Guide](installation.md) → Deploy ThreatCopilot
2. [Agent Configuration](agents.md) → Customize detection rules
3. [Deployment Guide](deployment.md) → Production deployment

### For DevOps Engineers
1. [Deployment Guide](deployment.md) → Production setup
2. [Architecture Deep Dive](architecture.md) → System design
3. [Installation Guide](installation.md) → Troubleshooting

---

## 📊 Key Diagrams

### System Architecture
```
┌─────────────────────────────────────────────────────────────┐
│                     📱 Vercel UI (Mobile)                    │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│                    FastAPI Backend                           │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│              LangGraph (3 AI Agents)                         │
│         ┌──────────┬──────────┬──────────┐                  │
│         │ Detector │ Analyzer │ Executor │                  │
│         └──────────┴──────────┴──────────┘                  │
└────────────────────────┬────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
┌────────▼────────┐ ┌───▼────┐ ┌───────▼────────┐
│ Ollama Llama3   │ │ PyOD ML│ │ Elasticsearch  │
└─────────────────┘ └────────┘ └────────────────┘
```

### 18-Second Response Timeline
```
0s  → Attack occurs
3s  → Logs indexed in Elasticsearch
6s  → ML detects anomaly (92% confidence)
8s  → AI identifies threat type (APT28)
10s → Playbook generated (7 steps)
15s → Phone notification sent
17s → Human approval received
18s → Threat neutralized (87% success)
```

---

## 🔧 Configuration Files

### Environment Variables
```bash
# Application
APP_NAME=ThreatCopilot
ENVIRONMENT=production

# Database
DATABASE_URL=postgresql://user:pass@localhost:5432/threatcopilot

# Elasticsearch
ELASTICSEARCH_HOST=localhost
ELASTICSEARCH_PORT=9200

# Ollama
OLLAMA_HOST=http://localhost:11434
OLLAMA_MODEL=llama3

# Security
SECRET_KEY=your-secret-key
JWT_ALGORITHM=HS256
```

### Docker Compose
```bash
# Start all services
docker-compose up -d

# View logs
docker-compose logs -f

# Stop services
docker-compose down
```

---

## 🛠️ Common Tasks

### Running Tests
```bash
# Unit tests
pytest tests/unit

# Integration tests
pytest tests/integration

# E2E tests
pytest tests/e2e

# Coverage report
pytest --cov=app tests/
```

### Database Migrations
```bash
# Create migration
alembic revision --autogenerate -m "description"

# Apply migrations
alembic upgrade head

# Rollback
alembic downgrade -1
```

### Updating ML Models
```bash
# Train new model
python scripts/train_model.py --data logs.csv --output models/detector.pkl

# Evaluate model
python scripts/evaluate_model.py --model models/detector.pkl --test test_data.csv

# Deploy model
cp models/detector.pkl /app/models/
docker-compose restart backend
```

---

## 📞 Support & Resources

### Getting Help
- **GitHub Issues**: [Report bugs or request features](https://github.com/yourusername/threatcopilot/issues)
- **Discussions**: [Ask questions and share ideas](https://github.com/yourusername/threatcopilot/discussions)
- **Email**: support@threatcopilot.ai
- **Slack**: [Join our community](https://threatcopilot.slack.com)

### Additional Resources
- **Blog**: [https://blog.threatcopilot.ai](https://blog.threatcopilot.ai)
- **YouTube**: [Video tutorials](https://youtube.com/@threatcopilot)
- **Twitter**: [@ThreatCopilot](https://twitter.com/threatcopilot)

---

## 🤝 Contributing

We welcome contributions! See our [Contributing Guide](../CONTRIBUTING.md) for:
- Code style guidelines
- Pull request process
- Development setup
- Testing requirements

---

## 📄 License

ThreatCopilot is licensed under the MIT License. See [LICENSE](../LICENSE) for details.

---

## 🎯 What's Next?

After reading the documentation:

1. **Try the Demo**: Run `docker-compose up -d` and visit `http://localhost:8000`
2. **Simulate an Attack**: `curl -X POST http://localhost:8000/api/v1/simulate-attack`
3. **View Dashboard**: Open `http://localhost:8000/dashboard` on your phone
4. **Customize Agents**: Edit `config/agents/*.yaml` files
5. **Deploy to Production**: Follow the [Deployment Guide](deployment.md)

---

