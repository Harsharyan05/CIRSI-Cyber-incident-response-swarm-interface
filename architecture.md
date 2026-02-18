# Architecture Deep Dive

> Comprehensive technical architecture of ThreatCopilot's AI-powered autonomous SOC

---

## Table of Contents

1. [System Overview](#system-overview)
2. [Architecture Layers](#architecture-layers)
3. [Component Details](#component-details)
4. [Data Flow](#data-flow)
5. [AI Agent System](#ai-agent-system)
6. [Security Architecture](#security-architecture)

---

## System Overview

ThreatCopilot implements a 5-layer autonomous pipeline that processes cyber threats from detection to neutralization in 18 seconds.

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        PRESENTATION LAYER                        │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   Web UI     │  │  Mobile UI   │  │  Admin Panel │         │
│  │  (Vercel)    │  │  (React)     │  │   (React)    │         │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘         │
│         │                  │                  │                  │
│         └──────────────────┴──────────────────┘                 │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│                         API GATEWAY LAYER                        │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │              FastAPI Backend (Port 8000)                │    │
│  │  • REST API Endpoints                                   │    │
│  │  • WebSocket for Real-time Updates                      │    │
│  │  • Authentication & Authorization                        │    │
│  │  • Rate Limiting & Request Validation                   │    │
│  └────────────────────────┬───────────────────────────────┘    │
└───────────────────────────┬─────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│                      AI ORCHESTRATION LAYER                      │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │              LangGraph Multi-Agent System               │    │
│  │                                                          │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐  │    │
│  │  │   Detector   │  │   Analyzer   │  │  Executor   │  │    │
│  │  │    Agent     │  │    Agent     │  │   Agent     │  │    │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬──────┘  │    │
│  │         │                  │                  │          │    │
│  │         └──────────────────┴──────────────────┘          │    │
│  └────────────────────────┬───────────────────────────────┘    │
└───────────────────────────┬─────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│                      INTELLIGENCE LAYER                          │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │ Ollama Llama3│  │   PyOD ML    │  │   tsfresh    │         │
│  │  (Local AI)  │  │  (Anomaly)   │  │  (Features)  │         │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘         │
└─────────┴──────────────────┴──────────────────┴─────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│                         DATA LAYER                               │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │ Elasticsearch│  │  PostgreSQL  │  │  Vuln-Bank   │         │
│  │  (Logs)      │  │  (Audit)     │  │  (Docker)    │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Architecture Layers

### 1. Presentation Layer

**Purpose**: User interfaces for threat monitoring and response approval

**Components**:
- Web Dashboard (Vercel-hosted)
- Mobile-optimized UI (React Native)
- Admin Control Panel
- Real-time notification system

**Technologies**:
- Frontend: React + TypeScript
- Styling: Tailwind CSS
- State Management: Zustand
- Real-time: WebSocket
- Deployment: Vercel Edge Network

### 2. API Gateway Layer

**Purpose**: Request routing, authentication, and API management

**Responsibilities**:
- RESTful API endpoints
- WebSocket connections for real-time updates
- JWT-based authentication
- Rate limiting (100 req/min per user)
- Request validation and sanitization
- API versioning (/api/v1/)

**Technologies**:
- Framework: FastAPI (Python 3.11+)
- Authentication: JWT + OAuth2
- Validation: Pydantic
- Documentation: OpenAPI/Swagger

### 3. AI Orchestration Layer

**Purpose**: Multi-agent coordination and decision-making

**Architecture**:
```
┌─────────────────────────────────────────────────────────┐
│              LangGraph Agent Orchestrator                │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌────────────────────────────────────────────────┐    │
│  │           Agent State Management               │    │
│  │  • Shared Memory                               │    │
│  │  • Context Passing                             │    │
│  │  • Decision History                            │    │
│  └────────────────────────────────────────────────┘    │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐  │
│  │   DETECTOR   │→ │   ANALYZER   │→ │  EXECUTOR   │  │
│  ├──────────────┤  ├──────────────┤  ├─────────────┤  │
│  │ • Log Parse  │  │ • Threat ID  │  │ • Playbook  │  │
│  │ • Anomaly    │  │ • Impact $   │  │ • Execute   │  │
│  │ • ML Score   │  │ • Chain Link │  │ • Monitor   │  │
│  │              │  │ • Priority   │  │ • Report    │  │
│  │ Time: 0-6s   │  │ Time: 6-10s  │  │ Time: 10-18s│  │
│  └──────────────┘  └──────────────┘  └─────────────┘  │
│                                                          │
│  ┌────────────────────────────────────────────────┐    │
│  │         Agent Communication Protocol            │    │
│  │  • Message Passing                              │    │
│  │  • State Synchronization                        │    │
│  │  • Error Handling & Retry                       │    │
│  └────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

**Agent Workflow**:
1. Detector Agent receives log → ML analysis → 92% confidence
2. Analyzer Agent assesses impact → $2.3M risk → APT28 pattern
3. Executor Agent generates playbook → awaits approval → executes

### 4. Intelligence Layer

**Purpose**: AI inference and machine learning processing

**Components**:

**a) Ollama Llama3 (Local LLM)**
- Model: Llama3-8B (quantized)
- Purpose: Natural language understanding, threat reasoning
- Inference time: ~2s per query
- Context window: 8K tokens
- Temperature: 0.3 (deterministic)

**b) PyOD Anomaly Detection**
- Algorithms: Isolation Forest, LOF, HBOS
- Training: 30-day rolling window
- Features: 47 extracted metrics
- Accuracy: 92% detection rate
- False positive: 5%

**c) tsfresh Feature Engineering**
- Time-series features: 789 calculated
- Window size: 5-minute intervals
- Feature selection: Top 47 by importance
- Processing time: <1s per log batch

### 5. Data Layer

**Purpose**: Persistent storage and log management

**Components**:

**a) Elasticsearch (Log Storage)**
```
Index Structure:
threat_logs_2024_01
├─ @timestamp
├─ source_ip
├─ destination_ip
├─ event_type
├─ severity
├─ raw_log
├─ ml_score
└─ agent_analysis

Retention: 90 days
Shards: 3 primary, 1 replica
Size: ~500GB/month
```

**b) PostgreSQL (Audit Trail)**
```
Schema:
├─ threats (id, type, severity, status, created_at)
├─ playbooks (id, threat_id, steps, approval_status)
├─ actions (id, playbook_id, action_type, result)
├─ users (id, username, role, permissions)
└─ audit_logs (id, user_id, action, timestamp)

Backup: Daily incremental, weekly full
Retention: 7 years (compliance)
```

**c) Vuln-Bank (Attack Simulation)**
- Dockerized vulnerable applications
- OWASP Top 10 scenarios
- Custom attack patterns
- Isolated network environment

---

## Component Details

### FastAPI Backend

**Architecture**:
```
app/
├── main.py                 # Application entry point
├── api/
│   ├── v1/
│   │   ├── endpoints/
│   │   │   ├── threats.py      # Threat detection endpoints
│   │   │   ├── playbooks.py    # Playbook management
│   │   │   ├── agents.py       # Agent control
│   │   │   └── auth.py         # Authentication
│   │   └── router.py
│   └── dependencies.py
├── core/
│   ├── config.py           # Configuration management
│   ├── security.py         # JWT, encryption
│   └── logging.py          # Structured logging
├── agents/
│   ├── detector.py         # Detector agent logic
│   ├── analyzer.py         # Analyzer agent logic
│   ├── executor.py         # Executor agent logic
│   └── orchestrator.py     # LangGraph orchestration
├── ml/
│   ├── anomaly_detector.py # PyOD models
│   ├── feature_extractor.py# tsfresh integration
│   └── model_trainer.py    # Training pipeline
├── models/
│   ├── threat.py           # SQLAlchemy models
│   ├── playbook.py
│   └── user.py
└── services/
    ├── elasticsearch.py    # ES client
    ├── ollama.py           # Ollama client
    └── notification.py     # Phone/email alerts
```

**Key Features**:
- Async/await for non-blocking I/O
- Connection pooling (20 connections)
- Request timeout: 30s
- Automatic retry with exponential backoff
- Circuit breaker pattern for external services

### LangGraph Agent System

**State Machine**:
```
┌─────────┐
│  START  │
└────┬────┘
     │
     ▼
┌─────────────┐
│  DETECTOR   │ ← Receives log entry
├─────────────┤
│ • Parse log │
│ • ML score  │
│ • Threshold │
└────┬────────┘
     │
     ▼ (if threat_score > 0.85)
┌─────────────┐
│  ANALYZER   │ ← Threat confirmed
├─────────────┤
│ • Identify  │
│ • Impact $  │
│ • Priority  │
└────┬────────┘
     │
     ▼ (if impact > $100K)
┌─────────────┐
│  EXECUTOR   │ ← High-priority threat
├─────────────┤
│ • Generate  │
│ • Approve?  │
│ • Execute   │
└────┬────────┘
     │
     ▼
┌─────────┐
│   END   │ ← Threat neutralized
└─────────┘
```

**Agent Communication**:
```python
# Shared state between agents
class ThreatState(TypedDict):
    log_entry: str
    ml_score: float
    threat_type: str
    impact_usd: int
    playbook_steps: List[str]
    approval_status: str
    execution_result: Dict
```

---

## Data Flow

### 18-Second Response Timeline

```
Time  | Layer              | Action                    | Output
------|--------------------|---------------------------|------------------
0s    | Data Layer         | Attack occurs             | Log generated
      |                    |                           |
3s    | Data Layer         | Elasticsearch indexes     | Log stored
      |                    |                           |
4s    | Intelligence       | ML feature extraction     | 47 features
      |                    |                           |
6s    | Intelligence       | PyOD anomaly detection    | 92% threat score
      | AI Orchestration   | Detector agent triggered  |
      |                    |                           |
8s    | Intelligence       | Ollama Llama3 analysis    | APT28 identified
      | AI Orchestration   | Analyzer agent processes  | $2.3M impact
      |                    |                           |
10s   | AI Orchestration   | Executor generates plan   | 7-step playbook
      |                    |                           |
15s   | API Gateway        | Push notification sent    | Phone alert
      | Presentation       | Judge views dashboard     |
      |                    |                           |
17s   | Presentation       | Judge taps APPROVE        | Approval received
      |                    |                           |
18s   | AI Orchestration   | Executor runs playbook    | Threat neutralized
      | Data Layer         | Audit log written         | 87% success
```

### Detailed Data Flow Diagram

```
┌──────────────┐
│ Attack Event │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────────────────────┐
│              Log Collection (0-3s)                    │
│  • Syslog                                             │
│  • Application logs                                   │
│  • Network traffic                                    │
│  • Endpoint telemetry                                 │
└──────┬───────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────┐
│         Elasticsearch Ingestion (3s)                  │
│  • Parse JSON/syslog format                           │
│  • Enrich with GeoIP, threat intel                    │
│  • Index in threat_logs_*                             │
└──────┬───────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────┐
│       Feature Extraction (3-4s)                       │
│  • tsfresh: 789 time-series features                  │
│  • Statistical: mean, std, entropy                    │
│  • Frequency: FFT, autocorrelation                    │
│  • Select top 47 features                             │
└──────┬───────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────┐
│       ML Anomaly Detection (4-6s)                     │
│  • PyOD Isolation Forest                              │
│  • Local Outlier Factor (LOF)                         │
│  • Histogram-based Outlier Score (HBOS)               │
│  • Ensemble voting → 92% confidence                   │
└──────┬───────────────────────────────────────────────┘
       │
       ▼ (if score > 0.85)
┌──────────────────────────────────────────────────────┐
│         Detector Agent (6-8s)                         │
│  • Validate ML score                                  │
│  • Extract IOCs (IPs, domains, hashes)                │
│  • Query threat intelligence feeds                    │
│  • Pass to Analyzer                                   │
└──────┬───────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────┐
│         Analyzer Agent (8-10s)                        │
│  • Ollama Llama3: Identify threat type                │
│  • MITRE ATT&CK mapping                               │
│  • Calculate financial impact                         │
│  • Determine priority (Critical/High/Medium)          │
│  • Pass to Executor                                   │
└──────┬───────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────┐
│         Executor Agent (10-15s)                       │
│  • Generate dynamic playbook                          │
│  • Steps: QUARANTINE → FORENSIC → ISOLATE            │
│  • Estimate success probability                       │
│  • Request human approval                             │
└──────┬───────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────┐
│       Human Approval (15-17s)                         │
│  • Push notification to phone                         │
│  • Display threat summary + playbook                  │
│  • Judge taps APPROVE or REJECT                       │
└──────┬───────────────────────────────────────────────┘
       │
       ▼ (if approved)
┌──────────────────────────────────────────────────────┐
│       Playbook Execution (17-18s)                     │
│  • Execute steps sequentially                         │
│  • Monitor success/failure                            │
│  • Write audit log to PostgreSQL                      │
│  • Send completion notification                       │
└──────┬───────────────────────────────────────────────┘
       │
       ▼
┌──────────────┐
│   Complete   │ → 87% threat neutralized ✅
└──────────────┘
```

---

## AI Agent System

### Agent Architecture

Each agent follows a consistent pattern:

```python
class BaseAgent:
    def __init__(self, llm, tools, memory):
        self.llm = llm              # Ollama Llama3
        self.tools = tools          # Available actions
        self.memory = memory        # Shared state
    
    async def process(self, state: ThreatState) -> ThreatState:
        # 1. Analyze current state
        analysis = await self.analyze(state)
        
        # 2. Make decision
        decision = await self.decide(analysis)
        
        # 3. Execute action
        result = await self.execute(decision)
        
        # 4. Update state
        return self.update_state(state, result)
```

### Detector Agent

**Responsibilities**:
- Parse incoming log entries
- Run ML anomaly detection
- Validate threat indicators
- Extract IOCs (Indicators of Compromise)

**Tools**:
- `parse_log()`: Extract structured data
- `ml_predict()`: Get anomaly score
- `query_threat_intel()`: Check known threats
- `extract_iocs()`: Find IPs, domains, hashes

**Decision Logic**:
```python
if ml_score > 0.85 and iocs_found:
    return "THREAT_DETECTED"
elif ml_score > 0.70:
    return "SUSPICIOUS"
else:
    return "BENIGN"
```

### Analyzer Agent

**Responsibilities**:
- Identify specific threat type
- Map to MITRE ATT&CK framework
- Calculate financial impact
- Determine response priority

**Tools**:
- `identify_threat()`: Use Ollama for classification
- `mitre_mapping()`: Map to ATT&CK techniques
- `calculate_impact()`: Estimate $ damage
- `prioritize()`: Assign urgency level

**Impact Calculation**:
```python
impact = (
    data_value * breach_probability +
    downtime_cost * duration_hours +
    regulatory_fine * compliance_risk +
    reputation_damage
)
```

### Executor Agent

**Responsibilities**:
- Generate dynamic response playbook
- Request human approval
- Execute remediation steps
- Monitor execution success

**Tools**:
- `generate_playbook()`: Create action steps
- `request_approval()`: Send phone notification
- `execute_step()`: Run remediation action
- `monitor_result()`: Track success/failure

**Playbook Example**:
```json
{
  "threat_id": "THR-2024-001",
  "steps": [
    {
      "order": 1,
      "action": "QUARANTINE_IP",
      "params": {"ip": "192.168.1.100"},
      "estimated_time": "2s"
    },
    {
      "order": 2,
      "action": "FORENSIC_DUMP",
      "params": {"source": "auth_logs", "timerange": "1h"},
      "estimated_time": "5s"
    },
    {
      "order": 3,
      "action": "VAULT_ISOLATION",
      "params": {"vault_id": "PROD_DB"},
      "estimated_time": "3s"
    }
  ],
  "estimated_success": 0.87
}
```

---

## Security Architecture

### Defense in Depth

```
┌─────────────────────────────────────────────────────┐
│              Layer 7: Audit & Compliance             │
│  • All actions logged to PostgreSQL                  │
│  • Immutable audit trail                             │
│  • SOC 2 Type II compliance                          │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│              Layer 6: Application Security           │
│  • Input validation (Pydantic)                       │
│  • SQL injection prevention (SQLAlchemy ORM)         │
│  • XSS protection (Content Security Policy)          │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│              Layer 5: Authentication & Authorization │
│  • JWT tokens (HS256)                                │
│  • Role-based access control (RBAC)                  │
│  • Multi-factor authentication (MFA)                 │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│              Layer 4: API Security                   │
│  • Rate limiting (100 req/min)                       │
│  • API key rotation                                  │
│  • Request signing                                   │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│              Layer 3: Network Security               │
│  • Docker network isolation                          │
│  • Firewall rules (iptables)                         │
│  • TLS 1.3 encryption                                │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│              Layer 2: Data Security                  │
│  • Encryption at rest (AES-256)                      │
│  • Encryption in transit (TLS)                       │
│  • Secure key management                             │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│              Layer 1: Infrastructure Security        │
│  • Container security scanning                       │
│  • Minimal base images (Alpine)                      │
│  • Regular security updates                          │
└─────────────────────────────────────────────────────┘
```

### Zero Trust Architecture

- No implicit trust between components
- Every request authenticated and authorized
- Least privilege access principle
- Continuous verification

### Data Privacy

- PII encryption in database
- Data retention policies (90 days logs, 7 years audit)
- GDPR compliance ready
- Data anonymization for ML training

---

## Performance Optimization

### Caching Strategy

```
┌─────────────────────────────────────────┐
│         Redis Cache Layer               │
├─────────────────────────────────────────┤
│ • Threat intelligence: 1 hour TTL       │
│ • ML model predictions: 5 min TTL       │
│ • User sessions: 30 min TTL             │
│ • API responses: 1 min TTL              │
└─────────────────────────────────────────┘
```

### Database Optimization

- Elasticsearch: 3 shards, 1 replica
- PostgreSQL: Connection pooling (20 connections)
- Indexes on frequently queried fields
- Partitioning by date (monthly)

### Scalability

**Horizontal Scaling**:
- FastAPI: 4 workers per instance
- Elasticsearch: Add nodes to cluster
- PostgreSQL: Read replicas

**Vertical Scaling**:
- Increase CPU for ML processing
- Increase RAM for Ollama inference
- SSD for faster I/O

---

## Monitoring & Observability

### Metrics Collection

```
┌─────────────────────────────────────────┐
│         Prometheus Metrics              │
├─────────────────────────────────────────┤
│ • Request latency (p50, p95, p99)       │
│ • Threat detection rate                 │
│ • Agent processing time                 │
│ • ML model accuracy                     │
│ • Database query time                   │
│ • Error rate by endpoint                │
└─────────────────────────────────────────┘
```

### Logging

- Structured JSON logs
- Log levels: DEBUG, INFO, WARNING, ERROR, CRITICAL
- Centralized logging to Elasticsearch
- Log retention: 90 days

### Alerting

- Slack/PagerDuty integration
- Alert on: High error rate, slow response, service down
- Escalation policy: L1 → L2 → L3

---

## Disaster Recovery

### Backup Strategy

- PostgreSQL: Daily full backup, hourly incremental
- Elasticsearch: Snapshot to S3 daily
- Configuration: Git version control
- Recovery Time Objective (RTO): 1 hour
- Recovery Point Objective (RPO): 1 hour

### High Availability

- Multi-AZ deployment
- Load balancer with health checks
- Automatic failover
- 99.9% uptime SLA

---

## Next Steps

- [Installation Guide](installation.md) - Set up ThreatCopilot
- [API Reference](api.md) - Explore API endpoints
- [Agent Configuration](agents.md) - Customize AI agents
- [Deployment Guide](deployment.md) - Deploy to production

