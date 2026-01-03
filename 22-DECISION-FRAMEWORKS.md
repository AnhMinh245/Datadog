# 22 - DECISION FRAMEWORKS

## 🎯 Mục Tiêu
Các framework để đưa ra quyết định về architecture, deployment, và configuration trong Datadog, đặc biệt cho banking context.

---

## 🏗️ Framework 1: Build vs Buy (Datadog vs Self-Hosted)

### **Decision Matrix**

```
┌─────────────────────────────────────────────────────────┐
│              BUILD (Self-Hosted)                         │
│           Prometheus + Grafana + ELK                     │
├─────────────────────────────────────────────────────────┤
│ Pros:                      │ Cons:                       │
│ ✓ Full control            │ ✗ High maintenance          │
│ ✓ Data stays on-premise   │ ✗ Team expertise required   │
│ ✓ No per-host cost        │ ✗ Scaling complexity        │
│ ✓ Customizable            │ ✗ Integration burden        │
│                           │ ✗ No enterprise support     │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                BUY (Datadog SaaS)                       │
├─────────────────────────────────────────────────────────┤
│ Pros:                      │ Cons:                       │
│ ✓ Quick setup             │ ✗ Higher cost at scale      │
│ ✓ All-in-one platform     │ ✗ Data leaves premise       │
│ ✓ Automatic updates       │ ✗ Vendor lock-in            │
│ ✓ Enterprise support      │ ✗ Less customization        │
│ ✓ Proven scalability      │ ✗ Pricing complexity        │
└─────────────────────────────────────────────────────────┘
```

### **Decision Tree**

```
START: Need observability platform
│
├─ Q1: Data sovereignty requirement?
│  ├─ YES → Must stay in-country?
│  │  ├─ YES, STRICT → BUILD or Hybrid
│  │  └─ YES, FLEXIBLE → Datadog EU site OK
│  └─ NO → Continue
│
├─ Q2: Team size & expertise?
│  ├─ < 5 DevOps engineers → BUY (Datadog)
│  └─ ≥ 10 DevOps engineers → Can BUILD
│
├─ Q3: Time to value?
│  ├─ Need NOW (< 1 month) → BUY
│  └─ Can wait (3-6 months) → Can BUILD
│
├─ Q4: Budget approval?
│  ├─ CapEx approved, OpEx restricted → BUILD
│  └─ OpEx OK → BUY (easier to justify)
│
└─ Q5: Compliance requirements?
   ├─ SOC 2, ISO required → BUY (certified)
   └─ Can self-certify → Can BUILD
```

### **TCO Comparison (5 Years)**

```
Self-Hosted (100 hosts):
├─ Infrastructure: $200K (servers, storage, network)
├─ Software: $50K (licenses if any)
├─ Engineering: $600K (3 FTE × $100K × 2 years setup + maintenance)
├─ Opportunity cost: $300K (features not built)
└─ Total: $1.15M

Datadog (100 hosts):
├─ Subscription: $600K (100 hosts × $1K/month × 60 months)
├─ Integration: $50K (initial setup)
└─ Total: $650K

Winner: Datadog ($500K savings)

BUT if data cannot leave premise → Must BUILD
```

### **Banking Decision**

**Typical Pattern:**
```
Tier 1 Banks (Large):
→ Hybrid Approach
  - Sensitive data: Self-hosted (ELK)
  - Infrastructure/Performance: Datadog
  - Cost: Higher, but compliant

Tier 2/3 Banks (Medium/Small):
→ Datadog with data scrubbing
  - Mask PII before sending
  - EU site for GDPR compliance
  - Cost-effective, compliant
```

---

## 📊 Framework 2: Metric Type Selection

### **Decision Flow**

```
What are you measuring?
│
├─ Current State (can go up/down)
│  └─ Use: GAUGE
│     Examples: users_online, queue_size, temperature
│
├─ Events over time (counting occurrences)
│  ├─ Need per-second rate?
│  │  └─ Use: COUNT + .as_rate() in query
│  │     Examples: requests, errors, orders
│  └─ Just count?
│     └─ Use: COUNT
│
├─ Duration or Size (need percentiles)
│  ├─ Low volume (< 1000/sec)?
│  │  └─ Use: DISTRIBUTION
│  │     Why: Full percentile flexibility
│  │     Examples: request_duration, file_size
│  └─ High volume?
│     └─ Use: HISTOGRAM
│        Why: More cost-effective
│        Examples: high-traffic API latency
│
└─ Unique items (cardinality)
   └─ Use: SET (rare, usually use gauge with distinct count)
```

### **Decision Matrix**

```
┌──────────┬─────────┬────────┬──────────┬─────────┐
│ Use Case │ Gauge   │ Count  │ Histogram│ Distrib │
├──────────┼─────────┼────────┼──────────┼─────────┤
│ Current  │   ✅    │   ❌   │    ❌    │   ❌    │
│ Events   │   ❌    │   ✅   │    ❌    │   ❌    │
│ Duration │   ❌    │   ❌   │    ✅    │   ✅    │
│ Low Vol  │   N/A   │   N/A  │    ✅    │   ✅    │
│ High Vol │   N/A   │   N/A  │    ✅    │   ⚠️    │
│ Cost     │   $     │   $    │    $$    │   $$$   │
└──────────┴─────────┴────────┴──────────┴─────────┘
```

### **Common Mistakes**

```
❌ Using Gauge for events
Problem: Loses count information
Example: gauge('requests', 1) per request
→ Only sees "1" in dashboard, not total count

✅ Fix: Use increment('requests')

❌ Using Distribution for everything
Problem: 5-10x more expensive than needed
Example: distribution('simple_counter', 1)

✅ Fix: Use count('simple_counter')

❌ High cardinality in tags
Problem: Metric explosion, high cost
Example: gauge('metric', value, tags=[f'user_id:{user_id}'])
→ Million metrics for million users

✅ Fix: Use gauge('metric', value, tags=[f'user_tier:{tier}'])
→ Only 3-4 metrics (free, basic, premium)
```

---

## 🏷️ Framework 3: Tagging Strategy

### **Design Principles**

```
1. LOW CARDINALITY
   ✓ Limited unique values per tag
   ✓ Rule: < 1000 unique values per tag
   ✗ Avoid: user_id, request_id, timestamp

2. HIERARCHICAL
   ✓ env > service > component > instance
   ✓ Allows drill-down analysis

3. CONSISTENT
   ✓ Same tags across all services
   ✓ Standard names: "env" not "environment" or "stage"

4. MEANINGFUL
   ✓ Business value tags (customer_tier)
   ✓ Operational tags (deployment_version)
```

### **Tag Selection Framework**

```
For each potential tag, ask:

Q1: How many unique values?
    > 1000 → DON'T USE as tag (use in logs instead)
    
Q2: Can I group/bucket values?
    Example: Instead of user_id (million values)
            → user_tier (3 values: free, premium, enterprise)
            
Q3: Will I query/filter by this?
    YES → Use as tag
    NO → Don't add (reduce cardinality)
    
Q4: Is this for debugging or analytics?
    Debugging → Logs (high cardinality OK)
    Analytics → Tags (low cardinality required)
```

### **Banking Tagging Template**

```yaml
# Required (Unified Service Tagging)
env: production | staging | development
service: core-banking | internet-banking | mobile-api
version: v2.1.3

# Infrastructure
datacenter: dc1 | dc2
availability_zone: zone-a | zone-b
host: hostname

# Application
team: core-banking | digital-banking | payments
component: frontend | backend | database | integration
tier: web | app | data

# Banking Specific
channel: branch | internet | mobile | atm | call-center
transaction_type: deposit | withdrawal | transfer | inquiry
business_unit: retail | corporate | treasury | wealth
criticality: critical | high | medium | low
compliance_scope: pci | sox | gdpr | all

# Performance
cache_status: hit | miss
response_status: 2xx | 4xx | 5xx (grouped, not individual)
```

### **Cardinality Calculation**

```
Example service with tags:

env: 3 values (dev, staging, prod)
service: 20 services
region: 3 regions
status: 3 status groups (2xx, 4xx, 5xx)

Per metric cardinality:
= 3 × 20 × 3 × 3 = 540 custom metrics

Cost: 540 × $0.05 = $27/month per metric

If you add user_id (million values):
= 3 × 20 × 3 × 3 × 1,000,000 = 540M metrics
Cost: $27M/month per metric! ❌

Solution: Don't use user_id as tag
→ Log it instead, query by user_id in logs
```

---

## 🚀 Framework 4: Agent Deployment Model

### **Model Comparison**

```
┌────────────────────────────────────────────────────────┐
│ Model 1: Agent Per Host (Standard)                    │
├────────────────────────────────────────────────────────┤
│ Architecture:                                          │
│   Each server → 1 Datadog Agent                       │
│                                                        │
│ Pros:              │ Cons:                            │
│ ✓ Simple           │ ✗ Agent overhead per host        │
│ ✓ Full metrics     │ ✗ More agents to manage          │
│ ✓ Easy debug       │ ✗ Resource usage per host        │
│                                                        │
│ Use When: Default choice for most scenarios           │
│ Banking: ✅ Recommended                               │
└────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────┐
│ Model 2: Cluster Agent (Kubernetes)                   │
├────────────────────────────────────────────────────────┤
│ Architecture:                                          │
│   K8s Cluster → 1 Cluster Agent                       │
│               → Multiple Node Agents (DaemonSet)       │
│                                                        │
│ Pros:              │ Cons:                            │
│ ✓ Centralized      │ ✗ K8s specific                  │
│ ✓ Cluster metrics  │ ✗ More complex setup            │
│ ✓ Less API calls   │ ✗ Additional component          │
│                                                        │
│ Use When: Kubernetes deployments                      │
│ Banking: ✅ For cloud-native apps                    │
└────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────┐
│ Model 3: Agentless (API Polling)                      │
├────────────────────────────────────────────────────────┤
│ Architecture:                                          │
│   Datadog → Poll APIs → Your infrastructure           │
│                                                        │
│ Pros:              │ Cons:                            │
│ ✓ No agent install │ ✗ Limited metrics               │
│ ✓ Less maintenance │ ✗ No logs/traces                │
│                    │ ✗ Higher latency                 │
│                                                        │
│ Use When: Legacy systems, can't install agent         │
│ Banking: ⚠️ Last resort only                         │
└────────────────────────────────────────────────────────┘
```

### **Decision Tree**

```
What platform?
│
├─ Kubernetes
│  └─ Use: Cluster Agent + Node Agents (DaemonSet)
│     Config: Helm chart
│     Why: Best for K8s, cluster-level metrics
│
├─ VMs/Bare Metal
│  └─ Use: Agent per host
│     Config: Package manager (apt/yum)
│     Why: Standard, proven approach
│
├─ Docker (not K8s)
│  └─ Use: Agent container
│     Config: Docker compose
│     Why: Container-native, easy scaling
│
├─ Cloud Functions/Serverless
│  └─ Use: Serverless integration (no agent)
│     Config: Extension/layer
│     Why: Ephemeral, can't run agent
│
└─ Legacy/Mainframe
   └─ Use: Agentless (API polling) or proxy
      Why: Can't modify legacy systems
```

### **Banking Decision**

```
Typical Banking Infrastructure:

Core Banking (Legacy):
├─ Platform: AIX, Solaris, Mainframe
├─ Approach: Agentless + Proxy agent
└─ Why: Cannot install on legacy

Digital Banking (Modern):
├─ Platform: Linux VMs, Docker, K8s
├─ Approach: Standard agent per host / Cluster agent
└─ Why: Full control, full metrics

Hybrid Pattern (Common):
├─ Layer 1 (Core): Agentless monitoring
├─ Layer 2 (Middle): Agent per host
├─ Layer 3 (Digital): Cluster agent
└─ All connected to single Datadog account
```

---

## 📊 Framework 5: Sampling Strategy

### **What to Sample**

```
High Volume Data:
├─ Health check requests
├─ Background job logs (info level)
├─ High-frequency metrics (telemetry)
└─ Non-critical traces

Never Sample:
├─ Error traces (always 100%)
├─ Slow transactions (p99+)
├─ Security events
├─ Financial transactions
└─ Audit logs
```

### **Sampling Decision Matrix**

```
┌─────────────────┬──────────┬──────────┬─────────────┐
│ Data Type       │ Volume   │ Sample%  │ Reason      │
├─────────────────┼──────────┼──────────┼─────────────┤
│ Error traces    │ Any      │ 100%     │ Must catch  │
│ Slow traces     │ Any      │ 100%     │ Investigate │
│ Normal traces   │ Low      │ 100%     │ Affordable  │
│                 │ Medium   │ 10-20%   │ Balance     │
│                 │ High     │ 1-5%     │ Cost        │
│ Info logs       │ Any      │ 10%      │ Not needed  │
│ Debug logs      │ Any      │ 0%       │ Never prod  │
│ Security logs   │ Any      │ 100%     │ Compliance  │
│ Business logs   │ Any      │ 100%     │ Analytics   │
│ Health checks   │ High     │ 0%       │ Noise       │
└─────────────────┴──────────┴──────────┴─────────────┘
```

### **APM Sampling Configuration**

```yaml
# Priority Sampling (Recommended)
apm_config:
  analyzed_spans:
    # Critical: 100%
    payment-service|*: 1.0
    auth-service|*: 1.0
    
    # Errors: Always
    *|error: 1.0
    
    # Slow: Always (p95+)
    *|slow: 1.0
    
    # Normal: Sample based on volume
    web-api|*: 0.1        # 10% (high traffic)
    mobile-api|*: 0.2     # 20% (medium traffic)
    admin-api|*: 1.0      # 100% (low traffic)
    
    # Health checks: Never
    *|health: 0.0
```

### **Log Sampling Strategy**

```yaml
# Index Strategy
logs_config:
  # 100% Index
  - type: index
    filter: status:(error OR critical)
    
  - type: index
    filter: @event_type:(transaction OR security)
    
  # 10% Sample
  - type: index_with_sampling
    filter: status:info
    sample_rate: 0.1
    
  # 0% Index (Archive only)
  - type: exclude
    filter: status:debug
    
  - type: exclude
    filter: @endpoint:/health
```

---

## 💰 Framework 6: Cost Optimization

### **Cost Reduction Decision Tree**

```
High Datadog bill?
│
├─ Step 1: Identify top cost drivers
│  ├─ Check: Custom metrics count
│  ├─ Check: Indexed logs volume
│  ├─ Check: APM spans indexed
│  └─ Use: Cost Dashboard
│
├─ Step 2: Quick wins (do first)
│  ├─ Exclude health check logs
│  ├─ Sample info logs (10%)
│  ├─ Remove high-cardinality tags
│  └─ Impact: 30-50% cost reduction
│
├─ Step 3: Metric optimization
│  ├─ Identify high-cardinality metrics
│  ├─ Group tag values (status:2xx vs status:200,201,...)
│  ├─ Sample high-volume metrics
│  └─ Impact: 20-40% cost reduction
│
├─ Step 4: Log optimization
│  ├─ Index only errors + business logs
│  ├─ Archive rest to S3
│  ├─ Shorter retention (7 days vs 15)
│  └─ Impact: 40-60% cost reduction
│
└─ Step 5: APM optimization
   ├─ Adjust sampling rates
   ├─ Use trace metrics (always free)
   ├─ Index selectively
   └─ Impact: 30-50% cost reduction
```

### **Cost Optimization Matrix**

```
┌────────────────┬──────────┬───────────┬────────────┐
│ Optimization   │ Effort   │ Savings   │ Risk       │
├────────────────┼──────────┼───────────┼────────────┤
│ Exclude health │ Easy     │ 10-20%    │ None       │
│ Sample info    │ Easy     │ 30-40%    │ Low        │
│ Fix high card  │ Medium   │ 20-50%    │ None       │
│ Reduce logs    │ Medium   │ 40-60%    │ Medium     │
│ APM sampling   │ Medium   │ 30-50%    │ Medium     │
│ Shorter retain │ Easy     │ 10-30%    │ Low        │
└────────────────┴──────────┴───────────┴────────────┘

Start with: Easy + High Savings + Low Risk
```

---

## 🏦 Framework 7: Banking Deployment Decision

### **Data Sovereignty Decision**

```
Does data need to stay in-country?
│
├─ YES, STRICT (Vietnam, Thailand, China)
│  ├─ Option 1: Self-hosted only
│  │  └─ Cost: High, Control: Full
│  │
│  ├─ Option 2: Hybrid
│  │  ├─ Sensitive: Self-hosted (ELK)
│  │  ├─ Infrastructure: Datadog
│  │  └─ Cost: Very High, Control: Partial
│  │
│  └─ Option 3: Data scrubbing + Datadog EU
│     ├─ Mask PII at agent
│     ├─ Use EU site (GDPR compliant)
│     └─ Cost: Medium, Control: Partial
│
└─ NO or FLEXIBLE (US, EU banks)
   └─ Use Datadog directly
      ├─ US1 or EU site
      └─ Cost: Medium, Control: Vendor-managed
```

### **Compliance Requirement Matrix**

```
┌─────────────┬──────────┬───────────┬───────────────┐
│ Requirement │ Priority │ Datadog   │ Alternative   │
├─────────────┼──────────┼───────────┼───────────────┤
│ SOC 2       │ Critical │ ✅ Native │ Self-certify  │
│ ISO 27001   │ Critical │ ✅ Native │ Self-certify  │
│ PCI-DSS     │ Critical │ ✅ Support│ Complex       │
│ GDPR        │ High     │ ✅ EU site│ Doable        │
│ In-country  │ High     │ ⚠️ Depends│ ✅ Self-host │
│ Audit trail │ Critical │ ✅ Native │ Build custom  │
│ RBAC        │ Critical │ ✅ Native │ Build custom  │
│ SSO         │ High     │ ✅ Native │ Complex       │
└─────────────┴──────────┴───────────┴───────────────┘

If all ✅ → Datadog is viable
If many ⚠️ or ❌ → Consider alternatives
```

---

## 🤔 Framework 8: When to Create New Monitor

### **Monitor Decision Flowchart**

```
Detected an issue?
│
├─ Q1: Will users notice?
│  ├─ NO → Don't create monitor
│  │       (Internal metric, not user-facing)
│  └─ YES → Continue
│
├─ Q2: Can team take action?
│  ├─ NO → Don't create monitor
│  │       (Can't fix or irrelevant)
│  └─ YES → Continue
│
├─ Q3: Is it critical?
│  ├─ YES → Create alert (PagerDuty)
│  └─ NO → Continue
│
├─ Q4: Happens frequently?
│  ├─ YES (>1/week) → Create warning alert
│  └─ NO (<1/month) → Track in dashboard only
│
└─ Q5: Existing monitor covers this?
   ├─ YES → Update existing monitor
   └─ NO → Create new monitor
```

### **Alert Threshold Decision**

```
How to set threshold?

Step 1: Baseline
  - Current value? (e.g., 0.5% error rate)
  
Step 2: Acceptable degradation
  - 2x? 5x? 10x worse?
  - Banking typically: 2-3x
  
Step 3: Set warning (early signal)
  - Warning = Baseline × 2
  - Example: 1% error rate
  
Step 4: Set alert (needs action)
  - Alert = Baseline × 5
  - Example: 2.5% error rate
  
Step 5: Validate
  - Run simulation
  - Check false positive rate
  - Adjust if needed
```

---

## 📝 Tóm Tắt

### **Framework Quick Reference**

```
1. Build vs Buy
   → TCO analysis, data sovereignty, team size
   
2. Metric Type
   → State=Gauge, Events=Count, Duration=Distribution
   
3. Tagging
   → Low cardinality, hierarchical, consistent
   
4. Agent Deployment
   → Platform-based, default=agent-per-host
   
5. Sampling
   → Errors=100%, Normal=10%, Health=0%
   
6. Cost Optimization
   → Start with easy wins (health checks, sampling)
   
7. Banking Deployment
   → Data sovereignty drives decision
   
8. Monitor Creation
   → User-facing + actionable = create alert
```

### **Decision Principles**

```
✓ Start simple, iterate
✓ Data-driven decisions
✓ Cost vs value trade-offs
✓ Banking = security first
✓ Document decisions
✓ Review quarterly
```

---

## ➡️ Related Topics

- [02 - Core Concepts](02-KHAI-NIEM-CO-BAN.md) - Foundation
- [16 - Best Practices](16-BEST-PRACTICES.md) - Implementation patterns
- [21 - Banking Implementation](21-BANKING-IMPLEMENTATION.md) - Banking specifics

---

**📌 Ghi Chú Của Bạn**
```
(Decisions made, rationale, lessons learned)








```

