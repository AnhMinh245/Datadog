# 20 - QUICK REFERENCE & DECISION GUIDE

## 🚀 Purpose
Fast lookup for common decisions, patterns, and quick references. Not a command cookbook - a decision support tool.

---

## 🎯 Common Scenarios & Decisions

### **Scenario 1: "Which Metric Type Should I Use?"**

```
Decision Tree:

What are you measuring?
│
├─ Current state (can go up/down)
│  Example: active_users, queue_size, temperature
│  → Use: GAUGE
│    statsd.gauge('metric', value)
│
├─ Counting events
│  Example: requests, orders, errors
│  │
│  ├─ Need rate (per second)?
│  │  → Use: COUNT + .as_rate() in query
│  │    statsd.increment('requests')
│  │    Query: sum:requests{*}.as_rate()
│  │
│  └─ Just total count?
│     → Use: COUNT
│       statsd.increment('orders')
│       Query: sum:orders{*}.as_count()
│
├─ Duration or size (need percentiles)
│  Example: response_time, file_size
│  │
│  ├─ Low volume (< 1K/sec)?
│  │  → Use: DISTRIBUTION
│  │    statsd.distribution('response_time', ms)
│  │    Why: Full percentile flexibility
│  │
│  └─ High volume (> 1K/sec)?
│     → Use: HISTOGRAM
│       statsd.histogram('response_time', ms)
│       Why: More cost-effective
│
└─ Unique counts
   Example: unique_visitors, distinct_errors
   → Use: SET (rare, usually use gauge with count distinct)
     statsd.set('visitors', user_id)
```

---

### **Scenario 2: "How to Tag This Metric?"**

```
Decision Framework:

Q1: How many unique values?
    < 100 → OK to tag ✅
    100-1000 → Caution ⚠️
    > 1000 → DON'T TAG ❌ (use logs instead)
    
Q2: Can I group/bucket?
    user_id (million values) → user_tier (3 values) ✅
    specific_error_message → error_type ✅
    timestamp → hour_of_day ✅
    
Q3: Will I query by this?
    YES → Tag it ✅
    NO → Don't add (reduce cardinality) ✅
    
Q4: For debugging or analytics?
    Debugging → Logs (high cardinality OK)
    Analytics → Tags (low cardinality required)

Examples:

✅ GOOD Tags:
  env:production              (3 values: dev, staging, prod)
  service:payment-api         (20 services in system)
  status:success              (2-3 values)
  user_tier:premium           (3 tiers)
  endpoint:/api/users         (~100 endpoints)
  region:us-east-1            (5 regions)

❌ BAD Tags (High Cardinality):
  user_id:12345               (millions)
  request_id:abc123           (unique each request)
  timestamp:1234567890        (always changing)
  email:user@example.com      (millions)
  specific_error_msg          (thousands of variations)
```

---

### **Scenario 3: "Dashboard Widget Selection"**

```
What do you want to show?

┌─ Single current value → Query Value
│  Examples: Current CPU: 67%, Active users: 850
│
├─ Trend over time → Timeseries
│  Examples: Request rate, Memory usage, Response time
│
├─ Ranking (top/bottom) → Top List
│  Examples: Top 10 hosts by CPU, Slowest 5 endpoints
│
├─ Distribution → Distribution Widget
│  Examples: Latency percentiles (p50/p95/p99)
│
├─ Pattern across dimensions → Heatmap
│  Examples: Errors by service × time, Latency by region
│
├─ Live events → Log Stream
│  Examples: Real-time errors, Deployment logs
│
├─ Infrastructure overview → Host Map
│  Examples: CPU across all hosts (color-coded)
│
├─ Service dependencies → Service Map
│  Examples: Microservices topology (auto-generated from APM)
│
└─ Multiple metrics comparison → Table
   Examples: Host comparison (CPU, RAM, Disk), Service SLOs
```

---

### **Scenario 4: "Query Aggregation Choice"**

```
Choose aggregation based on use case:

avg:
  ✅ Most common, general purpose
  ✅ CPU usage, memory, response time
  ✅ Smooths out spikes
  Example: avg:system.cpu.user{*}

sum:
  ✅ Counts, requests, bytes
  ✅ Total across all hosts/services
  Example: sum:http.requests{*}

max:
  ✅ Peak values, worst case
  ✅ Detect spikes
  Example: max:system.cpu.user{*} by {host}
         → Find which host peaked

min:
  ✅ Lowest value, best case
  ✅ Available resources
  Example: min:system.disk.free{*}
         → Least free disk space

pXX (percentiles):
  ✅ Latency, performance metrics
  ✅ Better than average for user experience
  Example: p95:trace.http.request.duration{*}
         → 95% of requests faster than this
```

---

### **Scenario 5: "How Much to Sample?"**

```
Decision Matrix:

┌─────────────────┬──────────┬──────────┬─────────────┐
│ Data Type       │ Volume   │ Sample%  │ Reason      │
├─────────────────┼──────────┼──────────┼─────────────┤
│ Error traces    │ Any      │ 100%     │ Must catch  │
│ Slow traces     │ Any      │ 100%     │ Debug perf  │
│ Normal traces   │ Low      │ 100%     │ Affordable  │
│                 │ Medium   │ 10-20%   │ Balance     │
│                 │ High     │ 1-5%     │ Cost        │
│ Info logs       │ Any      │ 10%      │ Not critical│
│ Debug logs      │ Any      │ 0%       │ Never prod  │
│ Security logs   │ Any      │ 100%     │ Compliance  │
│ Business logs   │ Any      │ 100%     │ Analytics   │
│ Health checks   │ High     │ 0%       │ Noise       │
└─────────────────┴──────────┴──────────┴─────────────┘

Implementation:

APM Sampling (in datadog.yaml):
apm_config:
  analyzed_spans:
    # Critical: always
    payment-service|*: 1.0
    auth-service|*: 1.0
    
    # Errors: always
    *|error: 1.0
    
    # Normal: sample based on volume
    web-api|*: 0.1        # 10% high traffic
    mobile-api|*: 0.2     # 20% medium traffic
    admin-api|*: 1.0      # 100% low traffic
    
    # Health: never
    *|health: 0.0

Log Sampling (in pipeline):
- Index 100%: status:(error OR critical)
- Sample 10%: status:info
- Exclude: status:debug, @endpoint:/health
```

---

### **Scenario 6: "Cost Optimization - Where to Start?"**

```
Priority Order (Impact vs Effort):

1. QUICK WINS (Do First):
   
   ✅ Exclude health check logs
      Impact: 20-40% log cost reduction
      Effort: 5 minutes
      How: Exclusion filter: @endpoint:/health
   
   ✅ Sample info logs to 10%
      Impact: 30-50% log cost reduction
      Effort: 10 minutes
      How: Sampling rule in log pipeline
   
   ✅ Remove high-cardinality tags
      Impact: Can save 50-90% metric cost
      Effort: 30 minutes (identify + fix)
      Example: user_id tag → 1M metrics → $50K/month
               Remove tag → 100 metrics → $500/month

2. MEDIUM EFFORT (Do Second):
   
   ✅ APM sampling configuration
      Impact: 30-50% APM cost
      Effort: 1 hour
      How: Configure analyzed_spans
   
   ✅ Reduce log retention
      Impact: 10-30% log cost
      Effort: 30 minutes
      How: 7 days vs 15 days
   
   ✅ Disable unused integrations
      Impact: 5-15% overall
      Effort: 1 hour (audit + disable)

3. ONGOING OPTIMIZATION:
   
   ✅ Regular usage dashboard review
      Frequency: Weekly
      Look for: Unexpected spikes, new high-cost services
   
   ✅ Tag cardinality monitoring
      Frequency: Monthly
      Alert on: Tags with > 1000 unique values
   
   ✅ Cost attribution by team
      Purpose: Accountability, chargeback

Cost Dashboard Query Examples:
- Metric cardinality: Check Metrics Summary page
- Log volume by service: sum:datadog.estimated_usage.logs.ingested_bytes{*} by {service}
- APM spans by service: sum:datadog.estimated_usage.apm.ingested_spans{*} by {service}
```

---

### **Scenario 7: "Agent Deployment Model Choice"**

```
Platform-Based Decision:

VMs/Bare Metal:
  → Agent per host (standard)
  Install: Package manager (apt, yum)
  Config: /etc/datadog-agent/datadog.yaml

Docker (Single Host):
  → Datadog Agent container
  Install: docker run datadog/agent
  Config: Environment variables
  Volumes: /var/run/docker.sock, /proc/, /sys/fs/cgroup/

Kubernetes:
  → Cluster Agent + Node Agents (DaemonSet)
  Install: Helm chart (recommended)
  Config: values.yaml
  Why: Cluster-level metrics, efficient API usage

AWS Lambda:
  → Serverless monitoring (no agent)
  Install: Lambda extension or Forwarder
  Why: Ephemeral, can't run persistent agent

Legacy/Mainframe:
  → Agentless (API polling) or Proxy
  Why: Cannot modify legacy systems

Decision Summary:
┌──────────────────┬──────────────────┬─────────────┐
│ Platform         │ Model            │ Method      │
├──────────────────┼──────────────────┼─────────────┤
│ Linux/Windows VM │ Agent per host   │ Package mgr │
│ Docker           │ Agent container  │ Docker run  │
│ Kubernetes       │ Cluster + Daemon │ Helm chart  │
│ Lambda           │ Extension        │ Layer       │
│ Legacy           │ Agentless        │ API poll    │
└──────────────────┴──────────────────┴─────────────┘
```

---

### **Scenario 8: "Monitor Type Selection"**

```
What to alert on?

Metric threshold breach:
  → Metric Monitor
  Example: CPU > 80%
  Query: avg:system.cpu.user{*} > 80

Log pattern:
  → Log Monitor
  Example: More than 10 errors in 5 minutes
  Query: status:error service:payment-api
  Threshold: > 10 in 5 min

APM performance:
  → APM Monitor
  Example: p95 latency > 500ms
  Query: p95:trace.http.request.duration{service:api} > 500

Multiple conditions (AND/OR):
  → Composite Monitor
  Example: High CPU AND High Memory
  Formula: a && b

Change detection:
  → Change Monitor
  Example: Traffic increased 50% vs 1 hour ago
  Query: avg:http.requests{*} increases > 50%

Unusual pattern:
  → Anomaly Monitor
  Example: CPU behaving abnormally
  Algorithm: ML-based anomaly detection

Predict future issue:
  → Forecast Monitor
  Example: Disk will be full in 1 week
  Query: forecast(avg:disk.used{*}, 'linear', 1w) >= 90

Decision Tree:
┌─ Simple threshold → Metric Monitor
├─ Log pattern → Log Monitor
├─ Need AND/OR logic → Composite Monitor
├─ Detect change → Change Monitor
├─ Unusual behavior → Anomaly Monitor
└─ Predict future → Forecast Monitor
```

---

## 🔍 Query Quick Reference

### **Query Syntax Cheatsheet**

```bash
Format: <agg>:<metric>{<filters>} [by {<tags>}] [.function()]

# Basic
avg:system.cpu.user{*}
sum:http.requests{env:production}
max:database.connections{*} by {host}

# Filters
{env:production}                    # Single tag
{env:production,service:api}        # AND
{env:production OR env:staging}     # OR
{service:web-*}                     # Wildcard
{env:production,-service:test}      # NOT

# Functions
.as_rate()          # Count to rate (per second)
.as_count()         # Rate to count
.rollup(avg, 60)    # Average in 60s windows
timeshift(..., 86400)  # Compare to 1 day ago (86400s)
anomalies(..., 'basic', 2)  # Anomaly detection
forecast(..., 'linear', 1w)  # Forecast 1 week ahead
ewma_5(...)         # Exponential smoothing

# Arithmetic
(a - b) / a * 100   # Calculate percentage
a + b               # Sum metrics
a / b               # Divide metrics

# Examples
Error rate %:
(sum:http.errors{*} / sum:http.requests{*}) * 100

Free memory %:
(avg:system.mem.total{*} - avg:system.mem.used{*}) / avg:system.mem.total{*} * 100

Compare to yesterday:
avg:system.cpu.user{*}
timeshift(avg:system.cpu.user{*}, 86400)
```

---

### **Log Search Quick Reference**

```bash
# Basic search
service:web-api
status:error
env:production

# Combine (implicit AND)
service:web-api status:error

# Boolean operators
service:web-api AND status:error
status:(error OR warning)
service:web-api -status:info  # NOT

# Wildcards
service:web-*
message:*timeout*

# Numeric ranges
@http.response_time:[100 TO 500]
@http.response_time:>1000
@user.age:<18

# Facets
@user.id:12345
@http.status_code:500
-@error.type:*  # Doesn't have error.type

# Time
@timestamp:[now-1h TO now]

# Common patterns
# Find errors for specific user:
service:payment-api status:error @user.id:12345

# Find slow requests:
service:api @duration:>1000

# Find failed payments:
service:payment status:error @transaction.type:payment
```

---

## 🏷️ Tagging Best Practices

### **Unified Service Tagging (Required)**

```yaml
# Always include (standard across all services):
env: production
service: payment-api
version: v1.2.3

Why:
- env: Filter by environment
- service: Group by service
- version: Correlate with deployments

How to set:
# Agent config
tags:
  - env:production
  - service:payment-api
  - version:v1.2.3

# Environment variables
DD_ENV=production
DD_SERVICE=payment-api
DD_VERSION=v1.2.3

# APM automatic
DD_SERVICE=payment-api DD_ENV=production DD_VERSION=v1.2.3 ddtrace-run python app.py
```

---

### **Additional Recommended Tags**

```yaml
Infrastructure:
  datacenter: us-east-1
  availability_zone: us-east-1a
  instance_type: t3.large
  host: web-server-01

Team/Org:
  team: backend
  squad: payments
  cost_center: engineering

Business:
  criticality: high
  customer_tier: premium

Banking-Specific:
  channel: internet-banking
  transaction_type: payment
  compliance_scope: pci
  business_unit: retail
```

---

## 🚨 Troubleshooting Checklist

### **Agent Not Sending Data**

```bash
Step 1: Verify agent status
  sudo datadog-agent status
  → Look for "Forwarder" section, should say "Running"

Step 2: Check connectivity
  curl -v https://api.datadoghq.com
  → Should return 200 OK
  → If fails: Firewall/proxy issue

Step 3: Verify API key
  sudo datadog-agent config | grep api_key
  → Should match key in Datadog UI (Organization Settings → API Keys)

Step 4: Check site setting
  sudo datadog-agent config | grep site
  → US1: datadoghq.com
  → EU1: datadoghq.eu
  → Must match your login URL

Step 5: Check logs for errors
  sudo tail -f /var/log/datadog/agent.log
  → Look for ERROR or WARN lines

Step 6: Run diagnostics
  sudo datadog-agent diagnose
  → Auto-checks common issues

Step 7: Send flare (support bundle)
  sudo datadog-agent flare
  → Sends diagnostics to Datadog support
```

---

### **High Metric Cardinality**

```bash
Problem: Metric count exploding, high bill

Identify:
  1. Go to Metrics → Summary
  2. Sort by "# of Tags"
  3. Look for metrics with > 1000 timeseries

Root Cause:
  High-cardinality tag (user_id, request_id, timestamp)

Fix:
  Option 1: Remove high-cardinality tag
    ❌ tags: [user_id:12345]
    ✅ tags: [user_tier:premium]
  
  Option 2: Use logs instead
    Metrics: Aggregate data (user_tier, status)
    Logs: Specific data (user_id, request_id)
  
  Option 3: Bucket values
    ❌ response_time:245ms (unique each time)
    ✅ response_time_bucket:200-300ms (few buckets)

Prevention:
  - Review tag cardinality monthly
  - Alert on metrics with > 1000 timeseries
  - Code review: Check tags before deploying
```

---

### **Missing Logs from Containers**

```yaml
Checklist:

□ Agent logs enabled?
  DD_LOGS_ENABLED=true

□ Container log collection enabled?
  DD_LOGS_CONFIG_CONTAINER_COLLECT_ALL=true

□ Docker socket mounted?
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock:ro

□ Correct permissions?
  Agent container user must be in docker group

□ Container using stdout/stderr?
  → Agent only collects stdout/stderr, not file logs

□ Check agent status:
  docker exec dd-agent agent status
  → Look for "Logs Agent" section

□ Label filtering?
  Check if container excluded by label filters
  
□ Log volume limits?
  Check if hit rate limits (10MB/s per container)
```

---

## 💰 Cost Quick Reference

### **Pricing Summary (As of 2026)**

```
Infrastructure Monitoring:
  $15 per host per month

APM:
  $31 per host per month
  + $1.70 per million spans indexed

Log Management:
  $0.10 per GB ingested
  $1.27 per million log events indexed

Custom Metrics:
  $0.05 per metric per month

RUM:
  $1.50 per thousand sessions

Synthetic Monitoring:
  $5 per 10,000 API test runs

Network Performance Monitoring:
  $5 per host per month
```

---

### **Cost Calculation Examples**

```
Example 1: Small Startup (10 hosts)
├─ Infrastructure: 10 × $15 = $150
├─ APM: 10 × $31 = $310
├─ Logs: 100GB × $0.10 = $10
└─ Total: ~$470/month

Example 2: Medium Company (100 hosts)
├─ Infrastructure: 100 × $15 = $1,500
├─ APM: 100 × $31 = $3,100
├─ Logs: 1TB × $0.10 = $100
├─ Custom Metrics: 500 × $0.05 = $25
└─ Total: ~$4,725/month

Example 3: Banking (200 hosts, heavy monitoring)
├─ Infrastructure: 200 × $15 = $3,000
├─ APM: 200 × $31 = $6,200
├─ Logs: 5TB × $0.10 = $500
├─ Log indexing: 100M events × $1.27/M = $127
├─ Custom Metrics: 2000 × $0.05 = $100
├─ RUM: 500K sessions × $1.50/1K = $750
├─ Synthetics: 100K tests × $5/10K = $50
└─ Total: ~$10,727/month
```

---

## 🔗 Essential Links

### **Datadog Sites**
```
US1 (default): https://app.datadoghq.com
US3:           https://us3.datadoghq.com
US5:           https://us5.datadoghq.com
EU1:           https://app.datadoghq.eu
```

### **Documentation**
```
Main Docs:     https://docs.datadoghq.com/
API Docs:      https://docs.datadoghq.com/api/
Learning:      https://learn.datadoghq.com/
Status:        https://status.datadoghq.com/
Agent GitHub:  https://github.com/DataDog/datadog-agent
```

---

## 📝 Quick Decision Tables

### **When to Use Datadog vs Alternatives**

```
Choose Datadog if:
✅ Want quick setup (< 1 week to production)
✅ Small/medium team (< 20 DevOps engineers)
✅ All-in-one solution needed
✅ Have budget ($5K-$50K+/month)
✅ Need 24/7 support
✅ Cloud-native architecture

Choose Self-Hosted (Prometheus + Grafana + ELK) if:
✅ Large DevOps team (10+ engineers)
✅ Data must stay on-premise (strict sovereignty)
✅ Very tight budget
✅ Want full control & customization
✅ Have time to maintain (2-3 FTE ongoing)

Choose New Relic if:
✅ Simpler pricing preferred (user-based)
✅ Strong mobile APM need
✅ Less infrastructure monitoring needed
```

---

## 📊 Banking-Specific Quick Reference

### **Compliance Checklist**

```yaml
Data Sovereignty:
  □ Use EU site for GDPR (app.datadoghq.eu)
  □ Use US site for US data (app.datadoghq.com)
  □ Configure data scrubbing (mask PII)
  □ Document data flows

Security:
  □ API keys in secrets manager (not git)
  □ RBAC configured (least privilege)
  □ SSO enabled (SAML)
  □ Audit logs enabled
  □ IP whitelisting configured

Compliance Certifications:
  ✅ SOC 2 Type II
  ✅ ISO 27001
  ✅ PCI-DSS (Service Provider Level 1)
  ✅ HIPAA
  ✅ GDPR compliant

SLO Recommendations:
  Core Banking: 99.95% availability
  Payment API: 99.99% availability
  Mobile API: 99.9% availability
  Internal Tools: 99.5% availability
```

---

## 📝 Tóm Tắt

```
This is a REFERENCE, not a tutorial:
✓ Use for quick decisions
✓ Lookup common patterns
✓ Choose right tool for the job
✓ Troubleshoot issues

NOT for:
✗ Step-by-step learning (see main docs)
✗ Comprehensive understanding (see 01-19)
✗ Copy-paste without understanding
```

---

**🎯 Knowledge Base - Always Here When You Need It**

Keep this reference handy for fast lookups and smart decisions!

---

**📌 Your Personal Quick Notes**
```
(Add your own shortcuts, decisions, patterns)








```
