# Agent Configuration

> Complete guide to configuring and customizing ThreatCopilot's AI agents

---

## Table of Contents

1. [Agent Overview](#agent-overview)
2. [Detector Agent](#detector-agent)
3. [Analyzer Agent](#analyzer-agent)
4. [Executor Agent](#executor-agent)
5. [Custom Agents](#custom-agents)
6. [Agent Tuning](#agent-tuning)

---

## Agent Overview

ThreatCopilot uses three specialized AI agents orchestrated by LangGraph:

```
┌─────────────────────────────────────────────────────────┐
│              LangGraph Orchestrator                      │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐  │
│  │   DETECTOR   │→ │   ANALYZER   │→ │  EXECUTOR   │  │
│  ├──────────────┤  ├──────────────┤  ├─────────────┤  │
│  │ Log → Threat │  │ Threat → $   │  │ $ → Action  │  │
│  │ 0-6 seconds  │  │ 6-10 seconds │  │ 10-18 sec   │  │
│  └──────────────┘  └──────────────┘  └─────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### Agent Communication

Agents share state through a common data structure:

```python
class ThreatState(TypedDict):
    # Input
    log_entry: str
    timestamp: str
    source: str
    
    # Detector output
    ml_score: float
    threat_detected: bool
    iocs: Dict[str, List[str]]
    
    # Analyzer output
    threat_type: str
    severity: str
    impact_usd: int
    mitre_attack: List[str]
    
    # Executor output
    playbook_id: str
    playbook_steps: List[Dict]
    approval_status: str
    execution_result: Dict
```

---

## Detector Agent

### Purpose

Identifies potential threats from log entries using ML and rule-based detection.

### Configuration File

`config/agents/detector.yaml`

```yaml
detector:
  # Model configuration
  ml_model:
    type: "pyod_ensemble"
    algorithms:
      - isolation_forest
      - local_outlier_factor
      - histogram_based_outlier
    threshold: 0.85
    confidence_min: 0.70
  
  # Feature extraction
  features:
    extractor: "tsfresh"
    window_size: "5m"
    features_count: 47
    normalization: "standard_scaler"
  
  # Detection rules
  rules:
    - name: "brute_force"
      pattern: "failed.*login.*attempt"
      threshold: 5
      timewindow: "5m"
    
    - name: "sql_injection"
      pattern: "(union|select|drop|insert).*from"
      severity: "critical"
    
    - name: "port_scan"
      pattern: "connection.*refused"
      threshold: 100
      timewindow: "1m"
  
  # Performance
  timeout: 30
  max_retries: 3
  batch_size: 100
  
  # Logging
  log_level: "INFO"
  log_detections: true
```

### Python Configuration

```python
from app.agents import DetectorAgent
from app.ml import PyODEnsemble

# Initialize ML model
ml_model = PyODEnsemble(
    algorithms=['isolation_forest', 'lof', 'hbos'],
    threshold=0.85
)

# Configure detector
detector = DetectorAgent(
    llm=ollama_llm,
    ml_model=ml_model,
    threshold=0.85,
    timeout=30,
    rules_path='config/detection_rules.yaml'
)

# Custom detection rule
detector.add_rule({
    'name': 'custom_threat',
    'pattern': r'malicious.*activity',
    'severity': 'high',
    'action': 'alert'
})
```

### Detection Rules

Create custom rules in `config/detection_rules.yaml`:

```yaml
rules:
  - id: "RULE-001"
    name: "Brute Force Detection"
    description: "Detect brute force login attempts"
    pattern: "failed.*login"
    conditions:
      - field: "attempts"
        operator: ">"
        value: 5
      - field: "timewindow"
        operator: "<"
        value: "5m"
    severity: "high"
    mitre: ["T1110.001"]
    actions:
      - "quarantine_ip"
      - "alert_admin"
  
  - id: "RULE-002"
    name: "SQL Injection"
    description: "Detect SQL injection attempts"
    pattern: "(union|select|drop).*from"
    severity: "critical"
    mitre: ["T1190"]
    actions:
      - "block_request"
      - "forensic_dump"
```

### ML Model Training

Train custom anomaly detection models:

```python
from app.ml import train_detector

# Load training data
logs = load_logs(days=30)

# Train model
model = train_detector(
    data=logs,
    algorithms=['isolation_forest', 'lof'],
    contamination=0.05,  # 5% expected anomalies
    n_estimators=100
)

# Save model
model.save('models/custom_detector.pkl')

# Evaluate
metrics = model.evaluate(test_data)
print(f"Accuracy: {metrics['accuracy']}")
print(f"Precision: {metrics['precision']}")
print(f"Recall: {metrics['recall']}")
```

### Feature Engineering

Customize feature extraction:

```python
from app.ml import FeatureExtractor

extractor = FeatureExtractor(
    features=[
        'mean', 'std', 'min', 'max',
        'entropy', 'fft_coefficient',
        'autocorrelation', 'variance'
    ],
    window_size='5m',
    overlap=0.5
)

# Extract features from logs
features = extractor.transform(logs)
```

---

## Analyzer Agent

### Purpose

Analyzes detected threats to determine type, severity, and financial impact.

### Configuration File

`config/agents/analyzer.yaml`

```yaml
analyzer:
  # LLM configuration
  llm:
    model: "llama3"
    temperature: 0.3
    max_tokens: 2000
    timeout: 30
  
  # Threat classification
  classification:
    confidence_threshold: 0.80
    use_mitre_mapping: true
    threat_intel_sources:
      - "misp"
      - "otx"
      - "virustotal"
  
  # Impact calculation
  impact:
    base_values:
      data_breach: 5000000
      downtime_per_hour: 100000
      regulatory_fine: 1000000
      reputation_damage: 500000
    
    multipliers:
      critical: 2.0
      high: 1.5
      medium: 1.0
      low: 0.5
  
  # MITRE ATT&CK
  mitre:
    framework_version: "v13"
    tactics_mapping: true
    techniques_mapping: true
  
  # Performance
  timeout: 30
  max_retries: 3
  cache_ttl: 3600
```

### Python Configuration

```python
from app.agents import AnalyzerAgent
from app.integrations import MITREAttack, ThreatIntel

# Initialize integrations
mitre = MITREAttack(version='v13')
threat_intel = ThreatIntel(sources=['misp', 'otx'])

# Configure analyzer
analyzer = AnalyzerAgent(
    llm=ollama_llm,
    mitre=mitre,
    threat_intel=threat_intel,
    impact_calculator=custom_impact_calc
)

# Custom impact calculation
def custom_impact_calc(threat):
    base = 1000000
    severity_multiplier = {
        'critical': 5.0,
        'high': 2.0,
        'medium': 1.0,
        'low': 0.5
    }
    return base * severity_multiplier[threat.severity]

analyzer.set_impact_calculator(custom_impact_calc)
```

### Threat Classification

Define custom threat types:

```yaml
# config/threat_types.yaml
threat_types:
  - name: "brute_force_attack"
    description: "Multiple failed login attempts"
    severity: "high"
    mitre: ["T1110.001", "T1110.003"]
    indicators:
      - "failed login"
      - "authentication failure"
      - "invalid credentials"
    impact_base: 500000
  
  - name: "sql_injection"
    description: "SQL injection attempt"
    severity: "critical"
    mitre: ["T1190"]
    indicators:
      - "union select"
      - "drop table"
      - "'; --"
    impact_base: 2000000
  
  - name: "ddos_attack"
    description: "Distributed denial of service"
    severity: "critical"
    mitre: ["T1498", "T1499"]
    indicators:
      - "connection flood"
      - "syn flood"
      - "udp flood"
    impact_base: 5000000
```

### MITRE ATT&CK Mapping

```python
from app.integrations import MITREAttack

mitre = MITREAttack()

# Map threat to MITRE
mapping = mitre.map_threat(
    threat_type='brute_force_attack',
    indicators=['failed login', 'multiple attempts']
)

print(f"Tactics: {mapping['tactics']}")
print(f"Techniques: {mapping['techniques']}")
print(f"Mitigations: {mapping['mitigations']}")

# Output:
# Tactics: ['Credential Access']
# Techniques: ['T1110.001 - Password Guessing']
# Mitigations: ['M1036 - Account Use Policies']
```

### Threat Intelligence Integration

```python
from app.integrations import ThreatIntel

threat_intel = ThreatIntel(
    sources=['misp', 'otx', 'virustotal'],
    api_keys={
        'otx': 'your_otx_key',
        'virustotal': 'your_vt_key'
    }
)

# Check IP reputation
reputation = threat_intel.check_ip('192.168.1.100')
print(f"Malicious: {reputation['is_malicious']}")
print(f"Threat score: {reputation['score']}")
print(f"Sources: {reputation['sources']}")

# Check domain
domain_info = threat_intel.check_domain('malicious.com')

# Check file hash
hash_info = threat_intel.check_hash('abc123...')
```

---

## Executor Agent

### Purpose

Generates and executes response playbooks to neutralize threats.

### Configuration File

`config/agents/executor.yaml`

```yaml
executor:
  # Playbook generation
  playbook:
    llm_model: "llama3"
    temperature: 0.2
    max_steps: 10
    require_approval: true
    approval_timeout: 300
  
  # Available actions
  actions:
    - name: "quarantine_ip"
      type: "network"
      timeout: 5
      rollback: true
    
    - name: "block_domain"
      type: "network"
      timeout: 5
      rollback: true
    
    - name: "isolate_host"
      type: "endpoint"
      timeout: 10
      rollback: true
    
    - name: "forensic_dump"
      type: "investigation"
      timeout: 30
      rollback: false
    
    - name: "vault_isolation"
      type: "data"
      timeout: 15
      rollback: true
  
  # Execution
  execution:
    parallel: false
    continue_on_error: false
    rollback_on_failure: true
    max_retries: 3
  
  # Notifications
  notifications:
    phone:
      enabled: true
      provider: "twilio"
      number: "+1234567890"
    
    email:
      enabled: true
      smtp_host: "smtp.gmail.com"
      recipients: ["security@company.com"]
    
    slack:
      enabled: true
      webhook_url: "https://hooks.slack.com/..."
```

### Python Configuration

```python
from app.agents import ExecutorAgent
from app.actions import ActionRegistry

# Register custom actions
actions = ActionRegistry()

@actions.register('custom_action')
async def custom_action(params):
    # Your custom logic
    ip = params['ip']
    # Block IP in firewall
    result = await firewall.block_ip(ip)
    return {'success': True, 'blocked_ip': ip}

# Configure executor
executor = ExecutorAgent(
    llm=ollama_llm,
    actions=actions,
    require_approval=True,
    notification_service=notification_svc
)
```

### Custom Actions

Create custom remediation actions:

```python
# app/actions/custom_actions.py

from app.actions import BaseAction

class QuarantineIPAction(BaseAction):
    name = "quarantine_ip"
    description = "Quarantine malicious IP address"
    
    async def execute(self, params):
        ip = params['ip']
        
        # Add to firewall blocklist
        await self.firewall.add_rule(
            action='deny',
            source_ip=ip,
            protocol='all'
        )
        
        # Log action
        await self.audit_log.write({
            'action': 'quarantine_ip',
            'ip': ip,
            'timestamp': datetime.now()
        })
        
        return {
            'success': True,
            'ip': ip,
            'message': f'IP {ip} quarantined successfully'
        }
    
    async def rollback(self, params):
        ip = params['ip']
        await self.firewall.remove_rule(source_ip=ip)
        return {'success': True, 'message': f'Rollback complete for {ip}'}
```

### Playbook Templates

Define reusable playbook templates:

```yaml
# config/playbook_templates.yaml
templates:
  - name: "brute_force_response"
    description: "Response to brute force attacks"
    steps:
      - action: "quarantine_ip"
        params:
          ip: "{{source_ip}}"
      
      - action: "forensic_dump"
        params:
          source: "auth_logs"
          timerange: "1h"
      
      - action: "notify_admin"
        params:
          severity: "high"
          message: "Brute force attack detected from {{source_ip}}"
  
  - name: "data_breach_response"
    description: "Response to data breach"
    steps:
      - action: "isolate_host"
        params:
          host: "{{affected_host}}"
      
      - action: "vault_isolation"
        params:
          vault_id: "{{vault_id}}"
      
      - action: "forensic_dump"
        params:
          source: "all_logs"
          timerange: "24h"
      
      - action: "notify_legal"
        params:
          severity: "critical"
```

### Approval Workflow

Configure approval requirements:

```python
from app.agents import ApprovalWorkflow

workflow = ApprovalWorkflow(
    require_approval_for=['critical', 'high'],
    auto_approve_for=['low'],
    approval_timeout=300,  # 5 minutes
    escalation_chain=[
        'security_analyst@company.com',
        'security_manager@company.com',
        'ciso@company.com'
    ]
)

executor.set_approval_workflow(workflow)
```

---

## Custom Agents

### Creating a Custom Agent

```python
from app.agents import BaseAgent
from langchain.agents import AgentExecutor

class CustomAgent(BaseAgent):
    def __init__(self, llm, tools, **kwargs):
        super().__init__(llm, tools, **kwargs)
        self.name = "custom_agent"
    
    async def process(self, state: ThreatState) -> ThreatState:
        # Your custom logic
        result = await self.analyze(state)
        
        # Update state
        state['custom_field'] = result
        
        return state
    
    async def analyze(self, state):
        # Use LLM for analysis
        prompt = f"Analyze this threat: {state['threat_type']}"
        response = await self.llm.ainvoke(prompt)
        return response

# Register custom agent
from app.agents import AgentRegistry

registry = AgentRegistry()
registry.register('custom_agent', CustomAgent)
```

### Agent Tools

Provide tools to agents:

```python
from langchain.tools import Tool

# Define custom tool
def check_reputation(ip: str) -> dict:
    """Check IP reputation in threat intelligence"""
    # Your logic here
    return {'ip': ip, 'reputation': 'malicious'}

# Create tool
reputation_tool = Tool(
    name="check_reputation",
    func=check_reputation,
    description="Check IP reputation in threat intelligence databases"
)

# Add to agent
detector.add_tool(reputation_tool)
```

---

## Agent Tuning

### Performance Optimization

```yaml
# config/performance.yaml
performance:
  # Concurrency
  max_concurrent_agents: 10
  agent_pool_size: 20
  
  # Timeouts
  agent_timeout: 30
  llm_timeout: 20
  action_timeout: 60
  
  # Caching
  cache_enabled: true
  cache_ttl: 3600
  cache_size_mb: 1024
  
  # Batching
  batch_processing: true
  batch_size: 100
  batch_timeout: 5
```

### Monitoring

```python
from app.monitoring import AgentMonitor

monitor = AgentMonitor()

# Track agent performance
@monitor.track
async def process_threat(threat):
    result = await detector.process(threat)
    return result

# Get metrics
metrics = monitor.get_metrics('detector')
print(f"Avg processing time: {metrics['avg_time_ms']}ms")
print(f"Success rate: {metrics['success_rate']}")
print(f"Error rate: {metrics['error_rate']}")
```

### A/B Testing

```python
from app.testing import ABTest

# Test two detector configurations
ab_test = ABTest(
    name='detector_threshold_test',
    variant_a={'threshold': 0.85},
    variant_b={'threshold': 0.90},
    metric='false_positive_rate',
    duration_days=7
)

# Run test
results = await ab_test.run()
print(f"Winner: {results['winner']}")
print(f"Improvement: {results['improvement']}%")
```

---

## Next Steps

- [Installation Guide](installation.md) - Set up ThreatCopilot
- [Architecture Deep Dive](architecture.md) - Understand system design
- [API Reference](api.md) - Explore API endpoints
- [Deployment Guide](deployment.md) - Deploy to production

