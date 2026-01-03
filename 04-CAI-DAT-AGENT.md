# 04 - AGENT DEPLOYMENT KNOWLEDGE

## 🎯 Mục Tiêu
Hiểu deployment models, key decisions, gotchas khi deploy Datadog Agent. Knowledge để tự tin triển khai trong môi trường production.

---

## 📚 Bản Chất: What is Datadog Agent?

### **Agent Architecture**

```
┌─────────────────────────────────────────────────┐
│  Application/Infrastructure                     │
└─────────────────┬───────────────────────────────┘
                  │ Collects metrics/logs/traces
┌─────────────────▼───────────────────────────────┐
│  Datadog Agent (Local process)                  │
│  ├─ Core Agent (metrics, health)                │
│  ├─ Trace Agent (APM traces)                    │
│  ├─ Process Agent (process monitoring)          │
│  └─ Logs Agent (log forwarding)                 │
└─────────────────┬───────────────────────────────┘
                  │ HTTPS (TLS 1.2+)
┌─────────────────▼───────────────────────────────┐
│  Datadog Backend (SaaS)                         │
│  https://api.datadoghq.com                      │
└─────────────────────────────────────────────────┘
```

**Key Points:**
- Agent = lightweight process (~70MB RAM, 5% CPU typical)
- Runs locally on each host/container
- Push model: Agent → Datadog (not pull)
- Written in Go (cross-platform)
- Open source: https://github.com/DataDog/datadog-agent

---

## 🏗️ Deployment Models

### **Model 1: Agent Per Host (Standard)**

```
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│   Host 1      │  │   Host 2      │  │   Host 3      │
│  ┌─────────┐  │  │  ┌─────────┐  │  │  ┌─────────┐  │
│  │ Agent   │  │  │  │ Agent   │  │  │  │ Agent   │  │
│  └─────────┘  │  │  └─────────┘  │  │  └─────────┘  │
│   App/Service │  │   App/Service │  │   App/Service │
└───────────────┘  └───────────────┘  └───────────────┘
```

**When to use:**
- ✅ VMs (AWS EC2, GCP Compute, Azure VMs)
- ✅ Bare metal servers
- ✅ Traditional deployments

**Pros:**
- ✅ Simple setup
- ✅ Full host metrics
- ✅ Easy troubleshooting
- ✅ Standard approach

**Cons:**
- ⚠️ One agent per host = more processes
- ⚠️ Manual updates if not automated

**Configuration:**
```yaml
Key decisions:
- Hostname strategy (auto vs manual)
- Tags (env, service, version)
- Features enabled (logs, APM, processes)
- Resource limits
```

---

### **Model 2: Containerized Agent (Docker/Kubernetes)**

#### **Docker (Single Host)**

```
┌─────────────────────────────────────────────┐
│  Docker Host                                 │
│  ┌─────────────┐  ┌──────────┐  ┌────────┐ │
│  │ DD Agent    │  │ App 1    │  │ App 2  │ │
│  │ Container   │  │ Container│  │Container│ │
│  └─────────────┘  └──────────┘  └────────┘ │
│         │              │              │      │
│         └──────────────┴──────────────┘      │
│              Monitors all containers         │
└─────────────────────────────────────────────┘
```

**When to use:**
- ✅ Docker Compose stacks
- ✅ Single-host container deployments

**Key Considerations:**
```yaml
Volumes needed:
  /var/run/docker.sock:  Monitor Docker
  /proc/:                System metrics
  /sys/fs/cgroup/:       Container metrics

Environment variables:
  DD_LOGS_CONFIG_CONTAINER_COLLECT_ALL: true
  DD_APM_NON_LOCAL_TRAFFIC: true
  DD_DOCKER_LABELS_AS_TAGS: true
```

**Gotchas:**
- ⚠️ Socket permissions (agent needs docker group)
- ⚠️ Log collection needs explicit enable
- ⚠️ APM requires port mapping (8126)

---

#### **Kubernetes (Cluster)**

```
┌──────────────────────────────────────────────┐
│  Kubernetes Cluster                           │
│  ┌────────────────────────────────────┐      │
│  │  Cluster Agent (Deployment)        │      │
│  │  - Cluster-level metrics           │      │
│  │  - Kubernetes API communication    │      │
│  └────────────────────────────────────┘      │
│              │                                │
│  ┌───────────┴───────────────────────┐       │
│  ▼           ▼           ▼           ▼       │
│ Node 1     Node 2     Node 3     Node 4      │
│ [Agent]    [Agent]    [Agent]    [Agent]     │
│ DaemonSet  DaemonSet  DaemonSet  DaemonSet   │
└──────────────────────────────────────────────┘
```

**When to use:**
- ✅ Kubernetes deployments (always)
- ✅ OpenShift

**Architecture:**
```
Cluster Agent (1 per cluster):
├─ Collects cluster-level metrics
├─ Communicates with K8s API
├─ Reduces API server load
└─ Provides HPA custom metrics

Node Agents (DaemonSet - 1 per node):
├─ Collects node & pod metrics
├─ Forwards logs from containers
├─ APM traces from pods
└─ Reports to Cluster Agent
```

**Key Decisions:**
```yaml
Deployment method:
  Option 1: Helm chart (recommended)
    - Easiest, most flexible
    - Values file for config
    - Easy upgrades
    
  Option 2: Operator
    - GitOps friendly
    - Custom resources
    
  Option 3: Manual YAML
    - Full control
    - Complex to maintain

Cluster Agent:
  Enable? YES (always recommended)
  Metrics Provider? YES (for HPA with custom metrics)
  
Node Agents:
  useHostNetwork: true (recommended for APM)
  Resource limits: Important!
    memory: 512Mi limit, 256Mi request
    cpu: 500m limit, 200m request
```

**Gotchas:**
- ⚠️ RBAC permissions required (ServiceAccount, ClusterRole)
- ⚠️ Cluster Agent needs API token
- ⚠️ Network policies may block agent communication
- ⚠️ Resource limits too low → OOMKilled

---

### **Model 3: Agentless (API Polling)**

```
┌──────────────────────┐
│  Datadog Backend     │
│                      │
│  Polls APIs ↓        │
└──────────────────────┘
           │
    ┌──────┴──────┐
    ▼             ▼
┌────────┐  ┌────────┐
│ AWS    │  │ Azure  │
│ CloudWatch  GCP    │
└────────┘  └────────┘
```

**When to use:**
- ✅ Cloud services without agent (Lambda, S3, RDS)
- ✅ SaaS integrations (GitHub, Slack, PagerDuty)
- ⚠️ Legacy systems where agent can't be installed

**Pros:**
- ✅ No agent installation needed
- ✅ Works with managed services

**Cons:**
- ❌ Limited metrics (only what API provides)
- ❌ No logs/traces
- ❌ Higher latency (polling delay)
- ❌ No custom metrics

**Banking Use Case:**
```
Hybrid model:
├─ Core Banking (Legacy): Agentless + Proxy
│   → Can't modify mainframe
│   → Polls system APIs
│
├─ Middle Tier (Java): Agent per host
│   → Full metrics, APM, logs
│
└─ Digital Banking (K8s): Cluster Agent + DaemonSet
    → Modern, containerized
```

---

## 🔑 Critical Configuration Decisions

### **1. Hostname Strategy**

**Problem:**
```
Default behavior: Agent auto-detects hostname
Issue in containers: Hostname = container ID (gibberish)
Issue in cloud: Hostname changes when instance replaced
```

**Solutions:**
```yaml
Option 1: Auto (default)
  hostname: auto-detected
  Good for: Stable VMs
  Bad for: Containers, ephemeral instances

Option 2: FQDN
  hostname_fqdn: true
  Good for: Multiple datacenters
  Bad for: Short names preferred

Option 3: Custom
  hostname: "web-prod-01"
  Good for: Consistent naming
  Bad for: Manual management

Option 4: Tags only (recommended for containers)
  hostname: none
  tags: [service:web, env:prod, version:v1.2.3]
  Good for: K8s, serverless
```

**Best Practice:**
```yaml
VMs/Bare Metal:
  Use FQDN or custom hostname
  
Containers/K8s:
  Don't rely on hostname
  Use tags (env, service, version)
  Let Datadog aggregate by tags
```

---

### **2. Tagging Strategy**

**Critical Decision:**
```
Tags = How you filter/group/aggregate

Bad tagging:
❌ No tags → Can't filter
❌ High cardinality tags → Cost explosion
❌ Inconsistent tags → Can't correlate

Good tagging:
✅ Unified Service Tagging (env, service, version)
✅ Low cardinality
✅ Consistent across all agents
```

**Recommended Tags:**
```yaml
# Required (Unified Service Tagging)
env: production
service: payment-api
version: v1.2.3

# Infrastructure
datacenter: us-east-1
availability_zone: us-east-1a
instance_type: t3.large

# Team/Business
team: backend
cost_center: engineering
criticality: high

# Custom (banking example)
channel: internet-banking
transaction_type: payment
compliance_scope: pci
```

**Tag Sources:**
```yaml
# 1. Static (in config)
tags:
  - env:production
  - team:backend

# 2. Environment variables
DD_TAGS="env:prod service:api"

# 3. Auto-discovery
collect_ec2_tags: true
collect_gce_tags: true

# 4. Docker labels
DD_DOCKER_LABELS_AS_TAGS: '{"app":"service","version":"version"}'

# 5. Kubernetes annotations
DD_KUBERNETES_POD_LABELS_AS_TAGS: '{"app":"service"}'
```

---

### **3. Feature Enablement**

**Decisions:**
```yaml
What to enable?

✅ Infrastructure Monitoring (always)
  - System metrics (CPU, RAM, disk)
  - Network
  - Default integrations

🤔 Logs (if needed)
  logs_enabled: true
  Cost: $0.10/GB ingested
  Consideration: Volume can be high
  
🤔 APM (if APM needed)
  apm_config.enabled: true
  Cost: $31/host + per span indexed
  Consideration: Instrumentation needed
  
🤔 Process Monitoring
  process_config.enabled: true
  Cost: Included
  Consideration: CPU overhead
  
🤔 Network Performance Monitoring
  system_probe_config.enabled: true
  Cost: Separate SKU
  Consideration: Kernel requirements
```

**Banking Recommendation:**
```yaml
Core Banking Tier:
  ✅ Infrastructure monitoring
  ✅ Logs (compliance/audit)
  ✅ APM (critical transactions)
  ❌ Process monitoring (too noisy)

Digital Banking Tier:
  ✅ All features enabled
  ✅ RUM for frontend
  ✅ Synthetic monitoring
```

---

### **4. Network & Security**

**Connectivity Requirements:**
```
Agent needs HTTPS outbound access:

US1 Site:
- Metrics: https://api.datadoghq.com (443)
- Logs: https://http-intake.logs.datadoghq.com (443)
- Traces: https://trace.agent.datadoghq.com (443)
- Process: https://process.datadoghq.com (443)

EU Site:
- Metrics: https://api.datadoghq.eu (443)
- Logs: https://http-intake.logs.datadoghq.eu (443)
- ...
```

**Firewall Rules:**
```bash
# Allow outbound HTTPS
iptables -A OUTPUT -p tcp --dport 443 -j ACCEPT

# Allow APM traces from other containers
iptables -A INPUT -p tcp --dport 8126 -j ACCEPT

# Allow DogStatsD metrics
iptables -A INPUT -p udp --dport 8125 -j ACCEPT
```

**Proxy Setup:**
```yaml
# If behind corporate proxy
proxy:
  https: http://proxy.company.com:3128
  no_proxy:
    - 169.254.169.254  # AWS metadata
    - 169.254.170.2   # ECS metadata
```

**Security Hardening:**
```yaml
Banking requirements:

1. API Key Protection:
   ✅ Use secrets manager (AWS Secrets, Vault)
   ✅ Never commit to git
   ✅ Rotate periodically

2. TLS:
   ✅ Min TLS 1.2
   ✅ Verify certificates
   
3. RBAC:
   ✅ Least privilege (read-only where possible)
   ✅ Service account per team
   
4. Data Scrubbing:
   ✅ Mask PII before sending
   ✅ Log processing rules
   
5. Network Isolation:
   ✅ Private subnets
   ✅ VPC endpoints (AWS)
   ✅ Private link (Azure)
```

---

## ⚠️ Common Gotchas & Troubleshooting

### **1. Agent Not Sending Data**

**Symptoms:**
```
- Agent status shows OK
- No data in Datadog UI
- No errors in logs
```

**Troubleshooting Checklist:**
```bash
# 1. Verify API key
datadog-agent config | grep api_key
→ Wrong key? Update datadog.yaml

# 2. Check connectivity
curl -v https://api.datadoghq.com
→ Blocked? Check firewall/proxy

# 3. Verify site setting
datadog-agent config | grep site
→ Wrong site? app.datadoghq.com vs app.datadoghq.eu

# 4. Check agent logs
tail -f /var/log/datadog/agent.log
→ Look for connection errors

# 5. Run diagnostics
datadog-agent diagnose
→ Auto-checks common issues
```

---

### **2. High Cardinality Explosion**

**Symptoms:**
```
- Custom metric count exploding
- High bill
- Metrics not appearing
```

**Root Cause:**
```yaml
# Bad: High cardinality tag
tags:
  - user_id:12345  # ❌ Millions of unique values
  - request_id:abc123  # ❌ Every request unique
  - timestamp:1234567890  # ❌ Always changing

# Good: Low cardinality
tags:
  - user_tier:premium  # ✅ 3-4 values
  - endpoint:/api/users  # ✅ ~100 endpoints
  - status:success  # ✅ 2-3 values
```

**Fix:**
```
1. Identify high cardinality tags
   → Datadog UI: Metrics → Summary → sort by cardinality
   
2. Remove or bucket tags
   → user_id → user_tier
   → specific_error_message → error_type
   
3. Use logs for high cardinality data
   → Request IDs, user IDs → logs, not metrics
```

---

### **3. Resource Exhaustion**

**Symptoms:**
```
- Agent using 100% CPU
- OOMKilled in Kubernetes
- Slow metrics collection
```

**Causes & Fixes:**
```yaml
Issue 1: Too many integrations enabled
Fix: Disable unused checks
  conf.d/*/conf.yaml → rename to .example

Issue 2: Log volume too high
Fix: Filter logs at agent
  logs_config:
    processing_rules:
      - type: exclude_at_match
        pattern: "health_check"

Issue 3: APM span volume
Fix: Sampling
  apm_config:
    analyzed_spans:
      service-name|*: 0.1  # 10% sampling

Issue 4: Process agent overhead
Fix: Reduce frequency or disable
  process_config:
    enabled: "false"
```

---

### **4. Missing Container Logs**

**Symptoms:**
```
- Infrastructure metrics OK
- Container logs not appearing
```

**Checklist:**
```yaml
1. Logs enabled?
   DD_LOGS_ENABLED=true

2. Container collection enabled?
   DD_LOGS_CONFIG_CONTAINER_COLLECT_ALL=true

3. Docker socket mounted?
   volumes:
     - /var/run/docker.sock:/var/run/docker.sock:ro

4. Permissions?
   → Agent needs docker group membership

5. Stdout/stderr?
   → Only collects stdout/stderr, not file logs

6. Label filters?
   → Check if container excluded by filters
```

---

## 📊 Deployment Patterns

### **Pattern 1: Phased Rollout**

```
Phase 1: Pilot (Week 1-2)
├─ Deploy to 2-3 non-critical hosts
├─ Verify data flowing
├─ Create base dashboards
└─ Test alert sensitivity

Phase 2: Staging (Week 3-4)
├─ Deploy to all staging
├─ Enable APM
├─ Enable logs
└─ Test under load

Phase 3: Production (Week 5+)
├─ Deploy to 10% production
├─ Monitor cost & performance
├─ Adjust sampling/filtering
├─ Roll out to 50%, then 100%
└─ Final tuning

Banking approach: Extra cautious
→ Pilot → Dev → QA → UAT → Staging → Prod (10% → 50% → 100%)
```

---

### **Pattern 2: Blue/Green Agent Upgrade**

```
Issue: Agent upgrades can be risky

Strategy:
1. Deploy new agent version to "green" hosts
2. Monitor for issues (24 hours)
3. If stable, upgrade "blue" hosts
4. If issues, quick rollback

For Kubernetes:
→ Use DaemonSet update strategy: RollingUpdate
   maxUnavailable: 1
   → Upgrades 1 node at a time
```

---

### **Pattern 3: Multi-Region Deployment**

```
Banking requirement: Data sovereignty

Architecture:
Region 1 (US):
  → Datadog US1 site (app.datadoghq.com)
  → US customer data
  
Region 2 (EU):
  → Datadog EU site (app.datadoghq.eu)
  → EU customer data (GDPR)
  
Region 3 (APAC):
  → Option 1: US3 site (us3.datadoghq.com)
  → Option 2: Dual-send (both US & EU)

Key: Different API keys per region
```

---

## 💰 Cost Optimization

### **Cost Drivers:**
```
1. Host count: $15/host/month (infra) + $31 (APM)
2. Custom metrics: $0.05/metric/month
3. Logs ingested: $0.10/GB
4. Log indexing: $1.27/million events
5. APM spans: $1.70/million indexed
```

### **Optimization Strategies:**
```
1. Exclude ephemeral hosts
   → Short-lived CI runners don't need agent
   
2. Sample logs
   → Info logs: 10% sample
   → Error logs: 100%
   
3. Sample APM
   → Normal traces: 10%
   → Errors: 100%
   → Slow traces: 100%
   
4. Filter at agent
   → Don't send health checks
   → Don't send debug logs
   
5. Short retention
   → Logs: 7 days vs 30 days
   → Old logs to S3 (cheaper)
```

---

## 📝 Tóm Tắt

### **Key Decisions:**
```
1. Deployment Model
   → VMs: Agent per host
   → Containers: Containerized agent
   → K8s: Cluster Agent + DaemonSet
   
2. Tagging Strategy
   → Unified Service Tagging (env, service, version)
   → Low cardinality
   
3. Features
   → Infra: Always
   → Logs: If needed (cost consideration)
   → APM: For critical services
   
4. Security
   → API key in secrets manager
   → Data scrubbing for PII
   → Network isolation
```

### **Gotchas to Remember:**
```
❌ High cardinality tags → Cost explosion
❌ Missing socket mount → No container metrics
❌ Wrong site → No data
❌ No resource limits (K8s) → OOMKilled
❌ Forgot to enable logs → No logs
```

### **Banking Considerations:**
```
✅ Data sovereignty (EU site for GDPR)
✅ Data scrubbing (mask PII)
✅ Audit trail (log all agent actions)
✅ High availability (multiple agents per tier)
✅ Compliance (SOC 2, PCI-DSS)
```

---

## ➡️ Bước Tiếp Theo

**Related Topics:**
- [03 - Architecture](03-KIEN-TRUC-DATADOG.md) - Understand agent internals
- [05 - Dashboards](05-DASHBOARD.md) - Visualize agent data
- [22 - Decision Frameworks](22-DECISION-FRAMEWORKS.md) - Agent deployment decisions

---

**📌 Ghi Chú Của Bạn**
```
(Deployment decisions, configurations, lessons learned)








```
