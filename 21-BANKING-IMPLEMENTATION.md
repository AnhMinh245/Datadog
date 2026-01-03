# 21 - TRIỂN KHAI DATADOG CHO NGÂN HÀNG

## 🎯 Mục Tiêu
Hiểu rõ các yêu cầu đặc thù khi triển khai Datadog cho khách hàng ngân hàng, từ security, compliance đến architecture decisions.

---

## 🏦 Đặc Thù Ngân Hàng

### **Yêu Cầu Khác Biệt**

```
Banking vs. Typical SaaS:
────────────────────────────────────────────────
Security:         CRITICAL vs. Important
Compliance:       MANDATORY vs. Nice-to-have
Availability:     99.95%+ vs. 99%
Data Sensitivity: EXTREME vs. Moderate
Audit Trail:      REQUIRED vs. Optional
Cost:             Justified vs. Optimized
Vendor Due Diligence: 6-12 months vs. 1 month
```

### **Key Stakeholders**

```
Technical:
├─ Solution Architect (Your primary contact)
├─ Infrastructure Team (Server, network)
├─ Application Team (Developers)
├─ Database Team (DBA)
└─ Security Team (InfoSec, Compliance)

Non-Technical:
├─ IT Management (Budget approval)
├─ Risk & Compliance (Regulatory)
├─ Audit (Internal/External)
└─ Procurement (Vendor management)
```

---

## 🔐 Security Requirements

### **1. Data Sovereignty**

**Vấn đề:**
```
Dữ liệu ngân hàng KHÔNG ĐƯỢC rời khỏi quốc gia
→ Datadog là SaaS ở US/EU
→ Làm sao comply?
```

**Solutions:**

#### Option A: Datadog EU Site
```yaml
Site: app.datadoghq.eu
Data Location: Frankfurt, Germany
GDPR Compliant: Yes
Suitable for: European banks, some APAC banks

Configuration:
  site: datadoghq.eu
  
Limitation:
  - Vẫn là data ở nước ngoài (EU)
  - Không suitable cho strict data residency (VN, Thailand, etc)
```

#### Option B: Data Scrubbing
```yaml
Strategy: Remove sensitive data BEFORE sending to Datadog

Implementation:
  1. Log scrubbing at Agent level
  2. Metric tag filtering
  3. Trace attribute masking
  
Example - Scrub Credit Card:
  logs_config:
    processing_rules:
      - type: mask_sequences
        name: mask_credit_card
        pattern: \d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}
        replace_with: "****-****-****-****"
```

#### Option C: Hybrid Approach
```
Sensitive Data: Keep on-premise (ELK, Grafana)
  - Customer PII
  - Transaction details
  - Account numbers
  
Non-Sensitive: Send to Datadog
  - Infrastructure metrics
  - Application performance
  - Aggregated business metrics
  
Trade-off: Split visibility, more complex
```

### **2. Network Security**

**Requirements:**
```
✅ All traffic must be encrypted (TLS 1.2+)
✅ Outbound only (no inbound from internet)
✅ Through proxy/firewall
✅ Whitelisted IPs only
✅ Certificate pinning
```

**Implementation:**

```yaml
# Datadog Agent config with proxy
proxy:
  https: http://proxy.bank.internal:3128
  no_proxy:
    - 169.254.169.254  # AWS metadata
    - localhost
    - *.bank.internal

# Whitelist Datadog IPs
# Get latest list: https://ip-ranges.datadoghq.com/
Firewall rules:
  Allow outbound HTTPS (443) to:
    - api.datadoghq.com (US) / api.datadoghq.eu (EU)
    - *.agent.datadoghq.com
    - *.logs.datadoghq.com
```

### **3. Access Control (RBAC)**

**Enterprise RBAC Setup:**

```yaml
Roles Structure:

1. Super Admin (2-3 people max)
   - Full access
   - User management
   - Billing

2. Security Auditor
   - Read-only access to all
   - Audit logs
   - No modification rights

3. Team Admin (per squad)
   - Manage team dashboards
   - Create monitors for team services
   - Cannot access other teams

4. Developer (Read-only)
   - View dashboards
   - View logs/traces
   - No modification

5. Executive (Business view)
   - Business dashboards only
   - No technical details
   - No configuration access
```

**SSO/SAML Integration:**

```yaml
Identity Provider: 
  - Active Directory / Azure AD
  - Okta
  - ADFS

Benefits:
  ✅ Centralized authentication
  ✅ MFA enforcement
  ✅ Auto-provisioning/deprovisioning
  ✅ Audit trail

Setup:
  1. Configure SAML in Datadog
  2. Map AD groups to Datadog roles
  3. Test with pilot users
  4. Rollout to all users
  5. Disable password login
```

### **4. Audit & Compliance**

**Audit Logging:**

```yaml
Track ALL actions:
  - User login/logout
  - Configuration changes
  - Dashboard creation/modification
  - Monitor creation/modification
  - Data access (who viewed what)

Datadog Audit Trail Feature:
  - Available in Enterprise plan
  - 90 days retention (can export)
  - JSON format
  - Integrate with SIEM

Export to SIEM:
  1. Enable Audit Trail
  2. Configure webhook to SIEM
  3. Or export to S3 → ingest to SIEM
```

**Compliance Reports:**

```
Required Reports (quarterly/annually):

1. Availability Report
   - Uptime % per service
   - Incident summary
   - MTTR trends

2. Performance Report
   - Transaction latency trends
   - Throughput metrics
   - SLA compliance

3. Security Report
   - Failed login attempts
   - Suspicious activities
   - Access patterns

4. Change Report
   - Configuration changes
   - Who changed what when
   - Approval trails
```

---

## 🏗️ Architecture Decisions

### **1. Agent Deployment Model**

**Option A: Agent Per Host (Recommended)**
```yaml
Topology:
  Each server/VM → 1 Datadog Agent

Pros:
  ✅ Simple, standard approach
  ✅ Full metrics coverage
  ✅ Easy troubleshooting

Cons:
  ❌ More agents to manage
  ❌ Resource overhead on each host

Best for: Most banking scenarios
```

**Option B: Cluster Agent (Kubernetes)**
```yaml
Topology:
  Kubernetes cluster → 1 Cluster Agent
                    → Multiple Node Agents

Pros:
  ✅ Centralized cluster-level monitoring
  ✅ Reduced API calls
  ✅ Custom metrics from K8s API

Best for: Cloud-native banking apps
```

**Option C: Agentless (Rare)**
```yaml
Topology:
  No agent, API polling only

Pros:
  ✅ No agent to install/maintain

Cons:
  ❌ Limited metrics
  ❌ No logs/traces
  ❌ Higher latency

Best for: Legacy systems that can't install agents
```

### **2. Network Topology**

**Typical Banking Setup:**

```
┌─────────────────────────────────────────────┐
│         INTERNET (Datadog SaaS)             │
└────────────────┬────────────────────────────┘
                 │ HTTPS (443)
                 │
┌────────────────▼────────────────────────────┐
│           DMZ Zone                           │
│  ┌──────────────────────────────┐           │
│  │   Proxy / Forward Proxy      │           │
│  │  (Squid / Corporate Proxy)   │           │
│  └──────────────┬───────────────┘           │
└─────────────────┼───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│        Internal Network Zone                 │
│                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │  App     │  │  App     │  │  App     │  │
│  │  Server  │  │  Server  │  │  Server  │  │
│  │  + Agent │  │  + Agent │  │  + Agent │  │
│  └──────────┘  └──────────┘  └──────────┘  │
│                                              │
│  ┌──────────┐  ┌──────────┐                │
│  │ Database │  │ Database │                │
│  │ + Agent  │  │ + Agent  │                │
│  └──────────┘  └──────────┘                │
└──────────────────────────────────────────────┘

Agent Configuration:
  proxy:
    https: http://proxy.bank.local:3128
```

### **3. Tagging Strategy for Banking**

**Unified Service Tagging + Banking Tags:**

```yaml
Standard (Required):
  env: production / staging / uat / development
  service: core-banking / internet-banking / mobile-api
  version: v2.1.3

Infrastructure:
  datacenter: dc1 / dc2
  availability_zone: zone-a / zone-b
  host: hostname
  cluster: prod-cluster-01

Application:
  team: core-banking-team / digital-banking-team
  component: frontend / backend / database
  tier: web / app / data

Banking-Specific:
  channel: branch / internet / mobile / atm
  transaction_type: deposit / withdrawal / transfer
  business_unit: retail / corporate / treasury
  criticality: critical / high / medium / low
  compliance_scope: pci / sox / gdpr

Example for a service:
  env: production
  service: fund-transfer-api
  version: v1.5.2
  datacenter: dc1
  team: payments
  component: backend
  channel: internet
  transaction_type: transfer
  criticality: critical
  compliance_scope: pci,sox
```

**Tag Cardinality Control:**

```yaml
✅ Low Cardinality (Good):
  env: 4 values (dev, uat, staging, prod)
  service: 20-50 services
  datacenter: 2-3 datacenters
  team: 10-15 teams
  channel: 5 channels

❌ High Cardinality (Avoid in tags):
  customer_id: millions
  account_number: millions
  transaction_id: unlimited
  session_id: unlimited
  
→ Put these in logs/traces, NOT in metric tags
```

---

## 💰 Cost Management

### **Banking vs. Typical Pricing**

```
Typical Startup:
  5 hosts × $15 = $75/month
  Total: ~$200-500/month

Banking (Small):
  100 hosts × $15 = $1,500/month
  APM: 100 hosts × $31 = $3,100/month
  Logs: 500 GB/day × $0.10 = $1,500/month
  Enterprise features: +$X/month
  Total: ~$10,000-20,000/month

Banking (Large):
  1000+ hosts
  Total: $100,000-500,000/month

BUT: Cost is NOT the main concern for banks
     → Availability, security, compliance are
```

### **Cost Optimization Strategies**

**1. Metric Optimization:**

```yaml
# High-volume, low-value metrics
health_check.status: Sample 1% or disable
session.keep_alive: Aggregate, don't send each

# Medium-value metrics  
api.requests: Sample 10-20%
database.queries: Sample 10-20%

# Critical metrics (never sample)
transaction.*: 100%
payment.*: 100%
error.*: 100%
security.*: 100%
```

**2. Log Optimization:**

```yaml
Index Strategy:
  
Index 100% (searchable):
  - ERROR, CRITICAL logs
  - Transaction logs
  - Security events
  - Audit trail
  
Index 10% (sample):
  - INFO logs
  - Background job logs
  
Index 0% (archive only):
  - DEBUG logs
  - Health check logs
  - Verbose application logs

Result: 70-80% cost reduction on logs
```

**3. APM Optimization:**

```yaml
Analyzed Spans Strategy:

100% sampling:
  - Payment transactions
  - Fund transfers
  - Login/Authentication
  - Critical APIs
  
10% sampling:
  - Inquiry APIs
  - Non-critical background jobs
  
1% sampling:
  - Health checks
  - Internal monitoring calls

Result: 60-70% cost reduction on APM
```

---

## 📊 Deployment Phases

### **Phase 1: POC (2-4 weeks)**

```yaml
Scope:
  - 5-10 non-production servers
  - 1-2 applications
  - Basic monitoring only

Goals:
  ✅ Prove technical feasibility
  ✅ Test network/security setup
  ✅ Evaluate data visibility
  ✅ Get stakeholder buy-in

Deliverables:
  - POC report
  - Architecture document
  - Cost estimation
  - Security assessment
```

### **Phase 2: Pilot (4-8 weeks)**

```yaml
Scope:
  - 1 production application (non-critical)
  - Full monitoring setup
  - Integration with existing tools

Goals:
  ✅ Production deployment
  ✅ Team training
  ✅ Process documentation
  ✅ Incident handling test

Deliverables:
  - Runbooks
  - Training materials
  - Standard dashboards
  - Alert playbooks
```

### **Phase 3: Production Rollout (3-6 months)**

```yaml
Scope:
  - All critical applications
  - Full infrastructure
  - Enterprise features

Approach: Phased rollout by criticality
  1. Low criticality (Month 1-2)
  2. Medium criticality (Month 2-4)
  3. High criticality (Month 4-6)

Per Application:
  Week 1: Agent deployment
  Week 2: APM instrumentation
  Week 3: Dashboard creation
  Week 4: Alert setup
  Week 5: Training & handover
```

### **Phase 4: Optimization (Ongoing)**

```yaml
Activities:
  - Cost optimization
  - Dashboard refinement
  - Alert tuning (reduce noise)
  - Advanced features adoption
  - Knowledge transfer

Metrics to Track:
  - MTTD (Mean Time To Detect)
  - MTTR (Mean Time To Resolve)
  - Alert accuracy (true vs false positives)
  - Dashboard usage
  - User adoption rate
```

---

## 📋 Deployment Checklist

### **Pre-Deployment**

```yaml
□ Business:
  □ Budget approved
  □ Stakeholders identified
  □ Success criteria defined
  □ Timeline agreed

□ Technical:
  □ Network access approved
  □ Firewall rules planned
  □ Proxy configured
  □ DNS resolution tested
  □ Certificate validated

□ Security:
  □ Security review completed
  □ Penetration test (if required)
  □ Data classification done
  □ RBAC roles designed
  □ Audit logging enabled

□ Compliance:
  □ Legal review (DPA, contract)
  □ Data residency confirmed
  □ Retention policies defined
  □ Audit requirements documented
```

### **During Deployment**

```yaml
□ Infrastructure:
  □ Agent installed on all hosts
  □ Agent status verified
  □ Metrics flowing
  □ Network connectivity confirmed

□ Application:
  □ APM instrumentation deployed
  □ Traces visible
  □ Logs collecting
  □ Custom metrics sending

□ Configuration:
  □ Tags standardized
  □ Integrations configured
  □ Dashboards created
  □ Monitors deployed

□ Security:
  □ SSO/SAML enabled
  □ Users provisioned
  □ Permissions validated
  □ Audit trail tested
```

### **Post-Deployment**

```yaml
□ Validation:
  □ E2E monitoring tested
  □ Alerts triggered (test)
  □ Dashboards accessible
  □ Performance acceptable

□ Documentation:
  □ Architecture documented
  □ Runbooks created
  □ Contact list updated
  □ Escalation procedures defined

□ Training:
  □ Admin training completed
  □ User training completed
  □ Knowledge transfer done

□ Handover:
  □ Operations team ready
  □ Support procedures defined
  □ Backup contacts identified
  □ Post-implementation review scheduled
```

---

## 🚨 Common Pitfalls & Solutions

### **1. Network/Connectivity Issues**

**Problem:**
```
Agent can't reach Datadog (most common issue)
```

**Checklist:**
```bash
# Test DNS resolution
nslookup api.datadoghq.com

# Test connectivity
curl -v https://api.datadoghq.com

# Test through proxy
curl -x http://proxy:3128 https://api.datadoghq.com

# Check agent logs
tail -f /var/log/datadog/agent.log

# Agent status
datadog-agent status
```

**Common causes:**
- Firewall blocking outbound 443
- Proxy misconfiguration
- Certificate issues
- Wrong Datadog site (US vs EU)

### **2. Permission Issues**

**Problem:**
```
Agent can't read logs / metrics
```

**Solution:**
```bash
# Agent runs as 'dd-agent' user
# Must have read permission

# For logs
chmod 644 /var/log/app/*.log
# OR add dd-agent to app group
usermod -a -G app dd-agent

# For database monitoring
GRANT SELECT ON performance_schema.* TO datadog@'localhost';
```

### **3. High Cardinality**

**Problem:**
```
Too many custom metrics → High cost
```

**Detection:**
```
Check metrics summary in Datadog:
Account Settings → Usage → Custom Metrics

Look for metrics with 1000+ combinations
```

**Solution:**
```python
# Bad: user_id in tags
statsd.increment('login', tags=[f'user_id:{user_id}'])
→ Millions of metrics!

# Good: user_tier instead
statsd.increment('login', tags=[f'user_tier:{user.tier}'])
→ Only 3-4 metrics (free, premium, enterprise)
```

### **4. Sensitive Data Leakage**

**Problem:**
```
PII in logs sent to Datadog
```

**Prevention:**
```yaml
# Agent-level scrubbing
logs_config:
  processing_rules:
    # Mask credit cards
    - type: mask_sequences
      name: mask_credit_card
      pattern: \d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}
      replace_with: "XXXX-XXXX-XXXX-XXXX"
    
    # Mask SSN
    - type: mask_sequences
      name: mask_ssn
      pattern: \d{3}-\d{2}-\d{4}
      replace_with: "XXX-XX-XXXX"
    
    # Mask emails
    - type: mask_sequences
      name: mask_email
      pattern: ([a-zA-Z0-9._%-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})
      replace_with: "user@masked.com"
```

---

## 📝 Tóm Tắt

### **Key Takeaways for Banking Implementation**

```
1. SECURITY FIRST
   - Data residency considerations
   - All traffic encrypted
   - RBAC properly configured
   - Audit trail enabled

2. COMPLIANCE IS MANDATORY
   - Know regulatory requirements
   - Document everything
   - Retain audit trails
   - Regular compliance reports

3. PHASED APPROACH
   - POC → Pilot → Rollout
   - Start with non-critical
   - Build confidence
   - Scale gradually

4. COST IS SECONDARY
   - Banks prioritize availability > cost
   - But still optimize when possible
   - Enterprise pricing negotiable

5. STAKEHOLDER MANAGEMENT
   - Many approvers
   - Long sales cycle (6-12 months)
   - Need executive buy-in
   - Security team is critical

6. CHANGE MANAGEMENT
   - Banks are risk-averse
   - Need extensive testing
   - Detailed documentation required
   - Training is essential
```

---

## ➡️ Next Steps

**Để triển khai thành công:**

1. **Hiểu khách hàng**
   - Regulatory requirements
   - Current monitoring setup
   - Pain points
   - Budget & timeline

2. **Thiết kế giải pháp**
   - Architecture design
   - Security controls
   - Compliance mapping
   - Cost estimation

3. **POC thành công**
   - Prove value quickly
   - Address security concerns
   - Get stakeholder buy-in

4. **Rollout có kế hoạch**
   - Phased approach
   - Risk mitigation
   - Rollback plan
   - Support structure

**Quay lại:** [README](README.md) để xem tổng quan

---

**📌 Ghi Chú Của Bạn**
```
(Implementation notes, customer-specific requirements, lessons learned)








```

