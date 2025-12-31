# 16 - BEST PRACTICES

## 🎯 Mục Tiêu
Tổng hợp các best practices để sử dụng Datadog hiệu quả, tối ưu chi phí và scale tốt.

---

## 🏷️ Tagging Strategy

### Unified Service Tagging

**Always use these 3 tags together:**
```
env:production
service:payment-api
version:v1.2.3
```

**Benefits:**
```
✅ Correlate metrics, logs, traces
✅ Filter by environment
✅ Track deployments
✅ Measure version performance
```

**Implementation:**

```yaml
# Agent config
tags:
  - env:production
  - service:payment-api
  - version:v1.2.3
```

```python
# In application
import os
from ddtrace import tracer

tracer.set_tags({
    'env': os.getenv('DD_ENV', 'development'),
    'service': os.getenv('DD_SERVICE', 'my-service'),
    'version': os.getenv('DD_VERSION', 'unknown')
})
```

```bash
# Environment variables
export DD_ENV=production
export DD_SERVICE=payment-api
export DD_VERSION=v1.2.3
```

### Tag Hierarchy

```
Level 1: Infrastructure
  - env: production, staging, development
  - region: us-east-1, eu-west-1
  - availability_zone: us-east-1a
  - instance_type: t3.large
  - host: web-server-01

Level 2: Application
  - service: payment-api
  - version: v1.2.3
  - component: backend, frontend, database
  - team: payments, platform, data

Level 3: Business
  - customer_tier: free, premium, enterprise
  - feature: new_checkout, legacy_checkout
  - experiment: variant_a, variant_b
```

### Tag Cardinality Rules

```
✅ LOW CARDINALITY (Good):
  - env: 3-5 values
  - service: 10-50 values
  - region: 5-10 values
  - team: 5-20 values
  
❌ HIGH CARDINALITY (Bad):
  - user_id: millions of values
  - request_id: unlimited
  - timestamp: unlimited
  - ip_address: thousands

RULE: If tag has > 1000 unique values → DON'T use as tag
→ Put in logs or span tags instead
```

---

## 📊 Metrics Best Practices

### 1. Choose Right Metric Type

```python
# ✅ Gauge for current state
statsd.gauge('queue.size', current_size)
statsd.gauge('users.active', active_count)

# ✅ Count for events
statsd.increment('orders.placed')
statsd.increment('errors.occurred')

# ✅ Histogram for distributions
statsd.histogram('request.duration', duration_ms)
statsd.histogram('file.size', file_bytes)

# ❌ Wrong: Using gauge for events
statsd.gauge('request.count', 1)  # BAD!

# ❌ Wrong: Using count for state
statsd.increment('queue.size')  # BAD!
```

### 2. Naming Conventions

```
Format: namespace.entity.attribute

✅ Good examples:
  web.requests.count
  database.query.duration
  cache.hit.rate
  payment.transaction.amount
  user.login.attempts

❌ Bad examples:
  requests (too vague)
  db_time (inconsistent format)
  PaymentAmount (wrong case)
  web-requests-count (wrong separator)
```

### 3. Metric Aggregation

```python
# ❌ Bad: Send individual metrics
for item in items:
    statsd.increment('items.processed')  # 1000 calls!

# ✅ Good: Aggregate then send
item_count = len(items)
statsd.increment('items.processed', item_count)  # 1 call

# ❌ Bad: High frequency
while True:
    statsd.gauge('current.value', value)
    time.sleep(0.1)  # Every 100ms!

# ✅ Good: Reasonable frequency
while True:
    statsd.gauge('current.value', value)
    time.sleep(10)  # Every 10s
```

### 4. Sample High Volume Metrics

```python
# High volume metric
for request in requests:  # Thousands per second
    statsd.increment('requests.total', sample_rate=0.1)
    
# Datadog multiplies by 10 to estimate true value
# Send 10% → Save 90% network/processing
```

---

## 🎨 Dashboard Best Practices

### 1. Dashboard Organization

```
Create dashboard hierarchy:

Executive Dashboards (High-level)
├── Business Metrics Overview
├── Overall System Health
└── Top-level SLOs

Team Dashboards (Medium-level)
├── Service Performance
├── Infrastructure Overview
└── Team SLOs

Detailed Dashboards (Low-level)
├── Service Deep Dive
├── Database Performance
└── Individual Component Metrics
```

### 2. Dashboard Design

```
Layout:
┌─────────────────────────────────────┐
│ MOST IMPORTANT (Top)                │
│ - Key business metrics              │
│ - Critical alerts                   │
│ - SLO status                        │
├─────────────────────────────────────┤
│ SUPPORTING METRICS (Middle)         │
│ - Request rates                     │
│ - Error rates                       │
│ - Latency                           │
├─────────────────────────────────────┤
│ DETAILS (Bottom)                    │
│ - Infrastructure metrics            │
│ - Detailed breakdowns               │
│ - Logs streams                      │
└─────────────────────────────────────┘

Rules:
✅ Most critical → Top
✅ 10-15 widgets max per dashboard
✅ Use template variables
✅ Add notes/context
✅ Consistent colors
```

### 3. Template Variables

```
Always use template variables for:

✅ Environment ($env)
✅ Service ($service)
✅ Region ($region)
✅ Host ($host)

Benefits:
- One dashboard for all envs
- Easy switching
- Reusable
- Less maintenance
```

---

## 🔍 APM Best Practices

### 1. Span Tags

```python
# ✅ Good: Add business context
span.set_tag('user.tier', 'premium')
span.set_tag('payment.amount', 99.99)
span.set_tag('cart.items', 5)
span.set_tag('discount.applied', True)

# ❌ Bad: High cardinality
span.set_tag('user.id', user_id)  # Millions of values!
span.set_tag('timestamp', timestamp)

# ❌ Bad: Sensitive data
span.set_tag('credit_card', card_number)
span.set_tag('password', password)
```

### 2. Sampling Strategy

```yaml
# Default: 100% trace metrics, 1% indexed traces
# Good for most cases

# High traffic services:
apm_config:
  analyzed_spans:
    web-api|*: 0.1  # Sample 10%
    web-api|slow: 1.0  # Always keep slow requests
    web-api|error: 1.0  # Always keep errors
    
# Low traffic, high value services:
apm_config:
  analyzed_spans:
    payment-api|*: 1.0  # Sample 100%
```

### 3. Resource Names

```python
# ✅ Good: Normalized resource names
# GET /users/123 → GET /users/?
# GET /products/456 → GET /products/?

from ddtrace import tracer

@app.route('/users/<user_id>')
def get_user(user_id):
    # Datadog automatically normalizes
    span = tracer.current_span()
    span.resource = 'GET /users/?'  # Manual if needed
    ...

# ❌ Bad: Unique resource names per request
span.resource = f'GET /users/{user_id}'  # Each user = different resource!
```

---

## 📝 Logging Best Practices

### 1. Structured Logging

```python
# ✅ Good: JSON structured
logger.info('Payment processed', extra={
    'event.type': 'payment',
    'payment.id': 'pay_123',
    'payment.amount': 99.99,
    'payment.currency': 'USD',
    'user.id': 'user_123',
    'user.tier': 'premium'
})

# ❌ Bad: Unstructured string
logger.info(f"User {user_id} paid ${amount} via {method}")
# Hard to search, parse, analyze
```

### 2. Log Levels

```python
# Use appropriate levels

# DEBUG: Development only, verbose
logger.debug(f"Variable x = {x}, y = {y}")

# INFO: Normal operations, business events
logger.info("User logged in", extra={'user.id': user_id})

# WARNING: Unusual but handled
logger.warning("Cache miss", extra={'key': cache_key})

# ERROR: Failures, requires attention
logger.error("Payment failed", extra={'error': str(e)})

# CRITICAL: System down, immediate action
logger.critical("Database unreachable")
```

### 3. Log Sampling

```python
import random

# Sample verbose logs
if log_level == 'INFO':
    if random.random() < 0.1:  # 10% sampling
        logger.info(message)
        
# Always log errors
elif log_level in ['ERROR', 'CRITICAL']:
    logger.error(message)
```

### 4. Exclusion Filters

```
Setup exclusion filters to reduce costs:

Filter 1: "Exclude Health Checks"
  Query: @http.url:"/health"
  Action: Exclude from indexing
  
Filter 2: "Exclude Debug Logs in Production"
  Query: status:debug env:production
  Action: Exclude from indexing
  
Filter 3: "Sample Info Logs"
  Query: status:info
  Action: Index 10%
  
Result: 60-80% cost reduction
```

---

## 🚨 Alerting Best Practices

### 1. Alert Hierarchy

```
P1 - CRITICAL (PagerDuty)
  - Service completely down
  - Major data loss
  - Security breach
  - Revenue impact
  → Page on-call immediately

P2 - HIGH (Slack + Email)
  - Partial service degradation
  - High error rate
  - Approaching capacity
  → Notify team, investigate soon

P3 - MEDIUM (Email)
  - Minor issues
  - Warnings
  - Non-critical alerts
  → Review during business hours

P4 - LOW (Dashboard only)
  - Informational
  - Trending issues
  → Review weekly
```

### 2. Alert on Symptoms, Not Causes

```
❌ Bad (Causes):
"MySQL connections = 100"
"Memory usage = 80%"
"Queue depth = 1000"

Why bad? Might be normal, not affecting users

✅ Good (Symptoms):
"API response time > 1s"
"Error rate > 1%"
"Checkout success rate < 95%"

Why good? Direct user impact, actionable
```

### 3. Composite Monitors

```yaml
# Instead of:
Monitor 1: CPU > 80%  → Alert
Monitor 2: Latency > 500ms → Alert
Monitor 3: Errors > 1% → Alert

# Result: 3 alerts for same issue

# Better:
Composite Monitor: "Service Degraded"
  Condition: (CPU > 80% AND Latency > 500ms) OR Errors > 5%
  
# Result: 1 alert, more accurate
```

### 4. Runbooks

```
Every alert must include:

Message template:
"""
🚨 {{name}}

**Current Value**: {{value}}
**Threshold**: {{threshold}}

**Impact**: [User impact description]

**Investigation**:
1. Check dashboard: {{link}}
2. View logs: [log query link]
3. Check recent deployments

**Resolution**:
1. [Step 1]
2. [Step 2]
3. [Step 3]

**Escalation**:
If not resolved in 15 minutes → Escalate to: @senior-engineer

**Runbook**: https://wiki/runbooks/alert-name
"""
```

---

## 💰 Cost Optimization

### 1. Custom Metrics

```
Calculation:
custom_metrics = unique(metric_name + tag_combinations)

Example:
Metric: api.requests
Tags: service (5 values), env (3), region (4)
= 5 × 3 × 4 = 60 custom metrics

Pricing: $0.05 per metric = $3/month

Optimization:
# Before: 60 metrics
tags: service, env, region, status_code (20 values)
= 5 × 3 × 4 × 20 = 1200 metrics = $60/month

# After: 60 metrics
tags: service, env, region, status_group (3: 2xx, 4xx, 5xx)
= 5 × 3 × 4 × 3 = 180 metrics = $9/month

Saved: $51/month (85%)
```

### 2. Log Indexing

```
Strategy:
1. Index only what you alert on
2. Archive everything to S3
3. Rehydrate when needed

Example:
100 GB logs/day ingested
- Index 10% (10 GB) = $10 ingestion + $127 indexing
- Archive 100% to S3 = $3/month storage
Total: $140/month

vs.

- Index 100% = $10 ingestion + $1270 indexing
Total: $1,280/month

Savings: $1,140/month (89%)
```

### 3. APM Trace Indexing

```
Default:
- 100% trace metrics (free)
- 1% indexed traces ($1.70 per million spans)

Optimize:
apm_config:
  analyzed_spans:
    # High traffic, low-value endpoints: 0.1%
    web-api|GET /health: 0.001
    
    # Important endpoints: 100%
    payment-api|*: 1.0
    
    # Always capture errors: 100%
    *|error: 1.0

Result: 70-90% cost reduction
```

### 4. Metric Types

```
Cost comparison:

Gauge: $0.05/metric
Count: $0.05/metric
Histogram: $0.05/metric × 5 aggregations = $0.25
Distribution: $0.05/metric × 10+ aggregations = $0.50+

Use Distribution only when you need:
✅ Query-time percentiles
✅ Aggregation across hosts
✅ Global percentiles

Otherwise use Histogram
```

---

## 🏗️ Infrastructure as Code

### Monitor as Code

```hcl
# Terraform
resource "datadog_monitor" "high_cpu" {
  name    = "High CPU Usage - {{host.name}}"
  type    = "metric alert"
  message = file("${path.module}/messages/high_cpu.tpl")
  
  query = <<EOT
avg(last_5m):avg:system.cpu.user{env:production} by {host} > 85
EOT

  thresholds = {
    critical = 85
    warning  = 70
  }
  
  notify_no_data    = true
  no_data_timeframe = 10
  
  tags = ["team:ops", "severity:high", "terraform:true"]
}
```

### Dashboard as Code

```json
{
  "title": "Service Overview - {{service}}",
  "widgets": [
    {
      "definition": {
        "type": "timeseries",
        "requests": [
          {
            "q": "avg:trace.http.request.duration{service:$service.name}",
            "display_type": "line"
          }
        ],
        "title": "Request Latency"
      }
    }
  ],
  "template_variables": [
    {
      "name": "service",
      "default": "*",
      "prefix": "service"
    }
  ]
}
```

---

## 📊 Monitoring Strategy

### What to Monitor

```
1. Golden Signals (Must have)
   ✅ Latency: How long requests take
   ✅ Traffic: How many requests
   ✅ Errors: How many failures
   ✅ Saturation: How full resources are

2. RED Method (Services)
   ✅ Rate: Requests per second
   ✅ Errors: Error rate
   ✅ Duration: Response time

3. USE Method (Resources)
   ✅ Utilization: % of resource used
   ✅ Saturation: Queue depth
   ✅ Errors: Resource errors

4. Business Metrics (Revenue)
   ✅ Orders per minute
   ✅ Revenue per hour
   ✅ Conversion rate
   ✅ Active users
```

### Monitoring Maturity Model

```
Level 1: Basic
✅ Infrastructure metrics (CPU, Memory, Disk)
✅ Basic alerts (service down)
✅ Manual investigation

Level 2: Intermediate
✅ APM traces
✅ Structured logs
✅ Custom metrics
✅ Alert on error rates
✅ Basic dashboards

Level 3: Advanced
✅ Full observability (metrics, logs, traces)
✅ Log correlation
✅ Business metrics
✅ SLOs/SLIs
✅ Anomaly detection
✅ Composite monitors

Level 4: Expert
✅ Proactive monitoring (forecasts)
✅ Auto-remediation
✅ Cost optimization
✅ Infrastructure as Code
✅ ML-based anomaly detection
```

---

## 📝 Tóm Tắt Checklist

### Daily Tasks
```
□ Review triggered monitors
□ Check SLO status
□ Review error logs
□ Check dashboard for anomalies
```

### Weekly Tasks
```
□ Review monitor performance
□ Check unused monitors
□ Review cost dashboard
□ Update runbooks
```

### Monthly Tasks
```
□ Monitor health review
□ Cost optimization review
□ Tag consistency audit
□ Dashboard cleanup
□ Update documentation
```

### Best Practices Summary

```
TAGGING:
✅ Use unified service tagging
✅ Low cardinality tags
✅ Consistent naming

METRICS:
✅ Right metric types
✅ Namespace.entity.attribute naming
✅ Aggregate before sending

DASHBOARDS:
✅ Important metrics on top
✅ Use template variables
✅ 10-15 widgets max

APM:
✅ Add business context
✅ Sample appropriately
✅ Normalize resource names

LOGGING:
✅ Structured JSON
✅ Appropriate log levels
✅ Exclude unnecessary logs

ALERTING:
✅ Alert on symptoms
✅ Composite monitors
✅ Include runbooks

COST:
✅ Reduce tag cardinality
✅ Index only what needed
✅ Sample high-volume data
```

---

## ➡️ Bước Tiếp Theo

**Bài tiếp theo**: [20 - Cheat Sheet](20-CHEAT-SHEET.md)

---

**📌 Ghi Chú Của Bạn**
```








```

