# API Reference

> Complete REST API documentation for ThreatCopilot

---

## Table of Contents

1. [Overview](#overview)
2. [Authentication](#authentication)
3. [Endpoints](#endpoints)
4. [WebSocket API](#websocket-api)
5. [Error Handling](#error-handling)
6. [Rate Limiting](#rate-limiting)

---

## Overview

### Base URL

```
Production:  https://api.threatcopilot.ai/api/v1
Development: http://localhost:8000/api/v1
```

### API Versioning

All endpoints are versioned with `/api/v1/` prefix. Breaking changes will increment the version number.

### Content Type

All requests and responses use `application/json` unless otherwise specified.

### Response Format

```json
{
  "success": true,
  "data": { },
  "message": "Operation completed successfully",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

---

## Authentication

### JWT Token Authentication

All protected endpoints require a JWT token in the Authorization header.

#### Login

```http
POST /api/v1/auth/login
Content-Type: application/json

{
  "username": "admin",
  "password": "SecurePassword123!"
}
```

**Response**:
```json
{
  "success": true,
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "token_type": "bearer",
    "expires_in": 1800
  }
}
```

#### Using Token

```http
GET /api/v1/threats
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

#### Refresh Token

```http
POST /api/v1/auth/refresh
Authorization: Bearer <refresh_token>
```

---

## Endpoints

### Health Check

#### GET /health

Check API health status.

**Request**:
```http
GET /health
```

**Response**:
```json
{
  "status": "healthy",
  "version": "1.0.0",
  "services": {
    "database": "connected",
    "elasticsearch": "connected",
    "ollama": "connected"
  },
  "uptime": 86400
}
```

---

### Threat Detection

#### POST /api/v1/detect

Detect threats from log entries.

**Request**:
```http
POST /api/v1/detect
Content-Type: application/json
Authorization: Bearer <token>

{
  "log_entry": "Failed login attempt from 192.168.1.100",
  "timestamp": "2024-01-15T10:30:00Z",
  "source": "auth_server",
  "metadata": {
    "user": "admin",
    "attempts": 5
  }
}
```

**Response**:
```json
{
  "success": true,
  "data": {
    "threat_id": "THR-2024-001",
    "threat_detected": true,
    "confidence": 0.92,
    "threat_type": "brute_force_attack",
    "severity": "high",
    "impact_usd": 2300000,
    "mitre_attack": ["T1110.001"],
    "recommended_actions": [
      "quarantine_ip",
      "alert_admin",
      "forensic_analysis"
    ],
    "processing_time_ms": 6000
  }
}
```

#### GET /api/v1/threats

List all detected threats.

**Query Parameters**:
- `page` (int): Page number (default: 1)
- `limit` (int): Items per page (default: 20, max: 100)
- `severity` (string): Filter by severity (low, medium, high, critical)
- `status` (string): Filter by status (detected, analyzing, mitigated, false_positive)
- `from_date` (ISO 8601): Start date
- `to_date` (ISO 8601): End date

**Request**:
```http
GET /api/v1/threats?page=1&limit=20&severity=high&status=detected
Authorization: Bearer <token>
```

**Response**:
```json
{
  "success": true,
  "data": {
    "threats": [
      {
        "id": "THR-2024-001",
        "type": "brute_force_attack",
        "severity": "high",
        "confidence": 0.92,
        "status": "mitigated",
        "impact_usd": 2300000,
        "detected_at": "2024-01-15T10:30:00Z",
        "mitigated_at": "2024-01-15T10:30:18Z"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 150,
      "pages": 8
    }
  }
}
```

#### GET /api/v1/threats/{threat_id}

Get detailed information about a specific threat.

**Request**:
```http
GET /api/v1/threats/THR-2024-001
Authorization: Bearer <token>
```

**Response**:
```json
{
  "success": true,
  "data": {
    "id": "THR-2024-001",
    "type": "brute_force_attack",
    "severity": "high",
    "confidence": 0.92,
    "status": "mitigated",
    "impact_usd": 2300000,
    "source_ip": "192.168.1.100",
    "target": "auth_server",
    "mitre_attack": ["T1110.001"],
    "timeline": [
      {
        "timestamp": "2024-01-15T10:30:00Z",
        "event": "threat_detected",
        "agent": "detector"
      },
      {
        "timestamp": "2024-01-15T10:30:08Z",
        "event": "threat_analyzed",
        "agent": "analyzer"
      },
      {
        "timestamp": "2024-01-15T10:30:15Z",
        "event": "playbook_generated",
        "agent": "executor"
      },
      {
        "timestamp": "2024-01-15T10:30:17Z",
        "event": "approval_received",
        "user": "judge@company.com"
      },
      {
        "timestamp": "2024-01-15T10:30:18Z",
        "event": "threat_mitigated",
        "success_rate": 0.87
      }
    ],
    "iocs": {
      "ips": ["192.168.1.100"],
      "domains": [],
      "hashes": []
    },
    "raw_log": "2024-01-15 10:30:00 auth_server: Failed login..."
  }
}
```

---

### Playbooks

#### GET /api/v1/playbooks

List all playbooks.

**Request**:
```http
GET /api/v1/playbooks?threat_id=THR-2024-001
Authorization: Bearer <token>
```

**Response**:
```json
{
  "success": true,
  "data": {
    "playbooks": [
      {
        "id": "PB-2024-001",
        "threat_id": "THR-2024-001",
        "status": "completed",
        "approval_status": "approved",
        "approved_by": "judge@company.com",
        "steps": 7,
        "success_rate": 0.87,
        "created_at": "2024-01-15T10:30:10Z",
        "completed_at": "2024-01-15T10:30:18Z"
      }
    ]
  }
}
```

#### GET /api/v1/playbooks/{playbook_id}

Get detailed playbook information.

**Response**:
```json
{
  "success": true,
  "data": {
    "id": "PB-2024-001",
    "threat_id": "THR-2024-001",
    "status": "completed",
    "steps": [
      {
        "order": 1,
        "action": "QUARANTINE_IP",
        "params": {"ip": "192.168.1.100"},
        "status": "completed",
        "result": "IP quarantined successfully",
        "duration_ms": 2000
      },
      {
        "order": 2,
        "action": "FORENSIC_DUMP",
        "params": {"source": "auth_logs", "timerange": "1h"},
        "status": "completed",
        "result": "Logs collected: 15,234 entries",
        "duration_ms": 5000
      }
    ],
    "estimated_success": 0.87,
    "actual_success": 0.87
  }
}
```

#### POST /api/v1/playbooks/{playbook_id}/approve

Approve a playbook for execution.

**Request**:
```http
POST /api/v1/playbooks/PB-2024-001/approve
Authorization: Bearer <token>

{
  "approved": true,
  "comment": "Approved for immediate execution"
}
```

**Response**:
```json
{
  "success": true,
  "data": {
    "playbook_id": "PB-2024-001",
    "status": "executing",
    "approved_by": "judge@company.com",
    "approved_at": "2024-01-15T10:30:17Z"
  }
}
```

---

### Agents

#### GET /api/v1/agents/status

Get status of all AI agents.

**Response**:
```json
{
  "success": true,
  "data": {
    "agents": [
      {
        "name": "detector",
        "status": "active",
        "processed_today": 1523,
        "avg_processing_time_ms": 3000,
        "success_rate": 0.98
      },
      {
        "name": "analyzer",
        "status": "active",
        "processed_today": 142,
        "avg_processing_time_ms": 2000,
        "success_rate": 0.95
      },
      {
        "name": "executor",
        "status": "active",
        "processed_today": 45,
        "avg_processing_time_ms": 5000,
        "success_rate": 0.87
      }
    ]
  }
}
```

#### POST /api/v1/agents/{agent_name}/configure

Configure agent parameters.

**Request**:
```http
POST /api/v1/agents/detector/configure
Authorization: Bearer <token>

{
  "threshold": 0.90,
  "timeout": 30,
  "retry_attempts": 3
}
```

---

### Analytics

#### GET /api/v1/analytics/dashboard

Get dashboard analytics.

**Response**:
```json
{
  "success": true,
  "data": {
    "today": {
      "threats_detected": 45,
      "threats_mitigated": 42,
      "false_positives": 3,
      "avg_response_time_s": 18,
      "total_impact_prevented_usd": 15000000
    },
    "this_week": {
      "threats_detected": 312,
      "threats_mitigated": 298,
      "success_rate": 0.95
    },
    "top_threats": [
      {"type": "brute_force_attack", "count": 15},
      {"type": "sql_injection", "count": 12},
      {"type": "ddos_attack", "count": 8}
    ]
  }
}
```

#### GET /api/v1/analytics/metrics

Get detailed metrics.

**Query Parameters**:
- `metric` (string): Metric name (response_time, detection_rate, etc.)
- `from_date` (ISO 8601): Start date
- `to_date` (ISO 8601): End date
- `interval` (string): Aggregation interval (hour, day, week)

**Response**:
```json
{
  "success": true,
  "data": {
    "metric": "response_time",
    "unit": "seconds",
    "data_points": [
      {"timestamp": "2024-01-15T00:00:00Z", "value": 17.5},
      {"timestamp": "2024-01-15T01:00:00Z", "value": 18.2}
    ],
    "statistics": {
      "min": 15.0,
      "max": 22.0,
      "avg": 18.0,
      "p50": 18.0,
      "p95": 20.5,
      "p99": 21.8
    }
  }
}
```

---

### Simulation

#### POST /api/v1/simulate-attack

Simulate an attack for testing.

**Request**:
```http
POST /api/v1/simulate-attack
Authorization: Bearer <token>

{
  "attack_type": "brute_force",
  "target": "auth_server",
  "intensity": "medium"
}
```

**Response**:
```json
{
  "success": true,
  "data": {
    "simulation_id": "SIM-2024-001",
    "attack_type": "brute_force",
    "status": "running",
    "estimated_duration_s": 30
  }
}
```

---

## WebSocket API

### Real-time Threat Updates

Connect to WebSocket for real-time threat notifications.

**Connection**:
```javascript
const ws = new WebSocket('ws://localhost:8000/ws/threats');

ws.onopen = () => {
  // Send authentication
  ws.send(JSON.stringify({
    type: 'auth',
    token: 'your_jwt_token'
  }));
};

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log('Threat update:', data);
};
```

**Message Format**:
```json
{
  "type": "threat_detected",
  "data": {
    "threat_id": "THR-2024-001",
    "severity": "high",
    "confidence": 0.92,
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

**Event Types**:
- `threat_detected`: New threat identified
- `threat_analyzed`: Analysis complete
- `playbook_generated`: Playbook ready for approval
- `approval_required`: Waiting for human approval
- `playbook_executing`: Executing remediation
- `threat_mitigated`: Threat neutralized

---

## Error Handling

### Error Response Format

```json
{
  "success": false,
  "error": {
    "code": "THREAT_NOT_FOUND",
    "message": "Threat with ID THR-2024-999 not found",
    "details": {}
  },
  "timestamp": "2024-01-15T10:30:00Z"
}
```

### HTTP Status Codes

| Code | Meaning | Description |
|------|---------|-------------|
| 200 | OK | Request successful |
| 201 | Created | Resource created |
| 400 | Bad Request | Invalid request parameters |
| 401 | Unauthorized | Missing or invalid authentication |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource not found |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server error |
| 503 | Service Unavailable | Service temporarily unavailable |

### Error Codes

| Code | Description |
|------|-------------|
| `INVALID_REQUEST` | Request validation failed |
| `AUTHENTICATION_FAILED` | Invalid credentials |
| `TOKEN_EXPIRED` | JWT token expired |
| `INSUFFICIENT_PERMISSIONS` | User lacks required permissions |
| `THREAT_NOT_FOUND` | Threat ID not found |
| `PLAYBOOK_NOT_FOUND` | Playbook ID not found |
| `AGENT_UNAVAILABLE` | AI agent not responding |
| `ML_MODEL_ERROR` | ML model inference failed |
| `DATABASE_ERROR` | Database operation failed |
| `RATE_LIMIT_EXCEEDED` | Too many requests |

---

## Rate Limiting

### Limits

| Endpoint | Rate Limit |
|----------|------------|
| `/api/v1/detect` | 100 requests/minute |
| `/api/v1/threats` | 200 requests/minute |
| `/api/v1/playbooks` | 200 requests/minute |
| `/api/v1/analytics` | 50 requests/minute |
| All other endpoints | 300 requests/minute |

### Rate Limit Headers

```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1642248600
```

### Rate Limit Exceeded Response

```json
{
  "success": false,
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded. Try again in 45 seconds.",
    "retry_after": 45
  }
}
```

---

## Code Examples

### Python

```python
import requests

# Authentication
response = requests.post(
    'http://localhost:8000/api/v1/auth/login',
    json={'username': 'admin', 'password': 'password'}
)
token = response.json()['data']['access_token']

# Detect threat
headers = {'Authorization': f'Bearer {token}'}
response = requests.post(
    'http://localhost:8000/api/v1/detect',
    headers=headers,
    json={
        'log_entry': 'Failed login from 192.168.1.100',
        'timestamp': '2024-01-15T10:30:00Z'
    }
)
threat = response.json()['data']
print(f"Threat detected: {threat['threat_type']}")
```

### JavaScript

```javascript
// Authentication
const login = async () => {
  const response = await fetch('http://localhost:8000/api/v1/auth/login', {
    method: 'POST',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify({
      username: 'admin',
      password: 'password'
    })
  });
  const data = await response.json();
  return data.data.access_token;
};

// Detect threat
const detectThreat = async (token) => {
  const response = await fetch('http://localhost:8000/api/v1/detect', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      log_entry: 'Failed login from 192.168.1.100',
      timestamp: '2024-01-15T10:30:00Z'
    })
  });
  const data = await response.json();
  console.log('Threat:', data.data.threat_type);
};
```

### cURL

```bash
# Login
curl -X POST http://localhost:8000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"password"}'

# Detect threat
curl -X POST http://localhost:8000/api/v1/detect \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "log_entry": "Failed login from 192.168.1.100",
    "timestamp": "2024-01-15T10:30:00Z"
  }'
```

---

## Next Steps

- [Installation Guide](installation.md) - Set up ThreatCopilot
- [Architecture Deep Dive](architecture.md) - Understand system design
- [Agent Configuration](agents.md) - Customize AI agents
- [Deployment Guide](deployment.md) - Deploy to production

