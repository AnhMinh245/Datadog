# 08 - LOG MANAGEMENT

## 🎯 Mục Tiêu Bài Học
Sau bài học này, bạn sẽ:
- Thu thập logs từ nhiều nguồn
- Parse và structure logs
- Search và filter logs hiệu quả
- Correlate logs với traces
- Create log-based alerts

---

## 📋 Log Management Overview

### Tại Sao Cần Log Management?

**Vấn Đề Truyền Thống:**
```
❌ Logs scattered across nhiều servers
❌ SSH vào từng server để xem logs
❌ Grep qua GB dữ liệu
❌ Không correlate được logs từ different services
❌ Khó troubleshoot distributed systems
```

**Giải Pháp Với Datadog:**
```
✅ Centralized log collection
✅ Real-time search trong seconds
✅ Structured logging với parsing
✅ Correlation với traces và metrics
✅ Log-based alerts và dashboards
```

### Log Pipeline

```
┌─────────────────────────────────────────────────────┐
│                  LOG PIPELINE                       │
└─────────────────────────────────────────────────────┘

1. COLLECTION
   │
   ├─ Application logs (via Agent)
   ├─ Container logs (Docker/K8s)
   ├─ Syslog
   ├─ Cloud provider logs (AWS CloudWatch, etc)
   ├─ HTTP API
   │
   ▼
2. PROCESSING
   │
   ├─ Parsing (JSON, Syslog, Custom)
   ├─ Attribute extraction
   ├─ Enrichment (GeoIP, user-agent, etc)
   ├─ Filtering (Exclusion filters)
   ├─ Sampling
   │
   ▼
3. INDEXING (Configurable)
   │
   ├─ Index filters decide what to index
   ├─ Indexed logs: Searchable, alertable
   ├─ Retention: 3-30 days (configurable)
   │
   ▼
4. ARCHIVE (All logs)
   │
   ├─ Send ALL logs to S3/GCS/Azure
   ├─ Long-term storage (compliance)
   ├─ Rehydrate when needed
   │
   ▼
5. QUERY & ALERT
   │
   └─ Search, analyze, alert on indexed logs
```

---

## 📥 Log Collection

### Method 1: File Tailing (Agent)

**Enable Logs in Agent:**
```yaml
# /etc/datadog-agent/datadog.yaml
logs_enabled: true
logs_config:
  container_collect_all: true
  processing_rules:
    - type: multi_line
      name: combine_stack_traces
      pattern: ^\d{4}-\d{2}-\d{2}
```

**Configure Log Source:**
```yaml
# /etc/datadog-agent/conf.d/app.d/conf.yaml
logs:
  - type: file
    path: /var/log/myapp/*.log
    service: my-app
    source: python
    sourcecategory: custom
    tags:
      - env:production
      - team:backend
```

**Multiple Log Files:**
```yaml
logs:
  - type: file
    path: /var/log/apache2/access.log
    service: apache
    source: apache
    
  - type: file
    path: /var/log/apache2/error.log
    service: apache
    source: apache
    log_processing_rules:
      - type: multi_line
        name: stack_trace
        pattern: ^\[
        
  - type: file
    path: /var/log/myapp/app.log
    service: my-app
    source: python
```

### Method 2: Docker Logs

**Agent Container:**
```bash
docker run -d \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  -v /proc/:/host/proc/:ro \
  -v /sys/fs/cgroup/:/host/sys/fs/cgroup:ro \
  -e DD_API_KEY=<API_KEY> \
  -e DD_LOGS_ENABLED=true \
  -e DD_LOGS_CONFIG_CONTAINER_COLLECT_ALL=true \
  datadog/agent:latest
```

**Container Labels:**
```yaml
# docker-compose.yml
services:
  web:
    image: myapp:latest
    labels:
      com.datadoghq.ad.logs: '[{
        "source": "nodejs",
        "service": "web-app",
        "tags": ["env:production"]
      }]'
```

### Method 3: Kubernetes

**DaemonSet Config:**
```yaml
# Helm values
datadog:
  logs:
    enabled: true
    containerCollectAll: true
    containerCollectUsingFiles: true
    
agents:
  volumes:
    - name: pointerdir
      hostPath:
        path: /var/lib/datadog-agent/logs
    - name: logpodpath
      hostPath:
        path: /var/log/pods
```

**Pod Annotations:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    ad.datadoghq.com/web.logs: '[{
      "source": "nodejs",
      "service": "web-api",
      "tags": ["env:production", "version:v1.2.3"]
    }]'
spec:
  containers:
  - name: web
    image: myapp:latest
```

### Method 4: HTTP API

**Python:**
```python
import requests
import json
import time

def send_log(api_key, message, **kwargs):
    url = 'https://http-intake.logs.datadoghq.com/v1/input'
    
    headers = {
        'Content-Type': 'application/json',
        'DD-API-KEY': api_key
    }
    
    log_entry = {
        'message': message,
        'ddsource': 'python',
        'ddtags': 'env:production,service:my-app',
        'hostname': 'my-server',
        'timestamp': int(time.time() * 1000),
        **kwargs
    }
    
    response = requests.post(url, headers=headers, json=log_entry)
    return response.status_code

# Usage
send_log(
    api_key='YOUR_API_KEY',
    message='User login successful',
    level='INFO',
    user_id=12345,
    ip_address='192.168.1.1'
)
```

### Method 5: Cloud Provider

**AWS CloudWatch:**
```
1. Setup Datadog AWS Integration
2. Enable Log Collection in integration settings
3. Select log groups to forward
4. Datadog automatically pulls logs
```

**Forwarder Lambda:**
```bash
# Deploy Datadog Forwarder
aws cloudformation deploy \
  --template-file datadog-forwarder.yaml \
  --stack-name datadog-forwarder \
  --parameter-overrides DdApiKey=<API_KEY>
```

---

## 📝 Log Format & Structure

### Structured Logging

**JSON Format (Recommended):**
```json
{
  "timestamp": "2026-01-01T10:30:15.123Z",
  "level": "ERROR",
  "service": "payment-api",
  "message": "Payment processing failed",
  "error": {
    "type": "StripeException",
    "message": "Card declined",
    "code": "card_declined"
  },
  "transaction": {
    "id": "txn_123456",
    "amount": 99.99,
    "currency": "USD"
  },
  "user": {
    "id": "user_12345",
    "email": "user@example.com",
    "tier": "premium"
  },
  "trace_id": "1234567890abcdef",
  "span_id": "abcdef1234567890",
  "env": "production",
  "version": "v1.2.3"
}
```

**Plain Text (Needs Parsing):**
```
2026-01-01 10:30:15 ERROR [payment-api] Payment processing failed - txn_123456 - Card declined - user_12345
```

### Standard Attributes

**Reserved Attributes:**
```
timestamp    - When log occurred
status       - Log level (info, error, etc)
service      - Service name
message      - Log message
host         - Hostname
source       - Technology (python, nginx, etc)
trace_id     - APM trace ID (for correlation)
span_id      - APM span ID
```

**Custom Attributes:**
```
Add any JSON fields:
- user.id
- transaction.id
- error.type
- http.method
- http.status_code
- duration
... anything relevant to your app
```

---

## 🔧 Log Parsing

### Grok Parser

**What is Grok:**
```
Pattern matching language to extract fields from logs
Based on regex, but with named patterns
```

**Example - Apache Access Log:**
```
Raw log:
192.168.1.1 - - [01/Jan/2026:10:30:15 +0000] "GET /api/users HTTP/1.1" 200 1234 "https://example.com" "Mozilla/5.0"

Grok Pattern:
%{ip:client_ip} - - \[%{date:timestamp}\] "%{word:http.method} %{notSpace:http.url} HTTP/%{number:http.version}" %{number:http.status_code} %{number:http.response_size} "%{notSpace:http.referer}" "%{data:http.user_agent}"

Result (Parsed):
{
  "client_ip": "192.168.1.1",
  "timestamp": "01/Jan/2026:10:30:15 +0000",
  "http": {
    "method": "GET",
    "url": "/api/users",
    "version": "1.1",
    "status_code": 200,
    "response_size": 1234,
    "referer": "https://example.com",
    "user_agent": "Mozilla/5.0"
  }
}
```

**Common Patterns:**
```
%{ip}            - IP address
%{date}          - Date/time
%{number}        - Number
%{word}          - Word (no spaces)
%{data}          - Any data
%{notSpace}      - Non-space characters
%{uuid}          - UUID
%{email}         - Email address
```

### Pipeline Processors

**1. Grok Parser:**
```
Extract fields using patterns
```

**2. Date Remapper:**
```
Convert string to timestamp
Set as official log timestamp
```

**3. Status Remapper:**
```
Map custom level field to official status
Example: "level" → "status"
```

**4. Service Remapper:**
```
Set service name from attribute
Example: "app_name" → "service"
```

**5. Category Processor:**
```
Add category based on attributes

Example:
If http.status_code >= 500
  → category: "server_error"
```

**6. Arithmetic Processor:**
```
Calculate new attributes

Example:
duration_seconds = duration_ms / 1000
```

**7. GeoIP Parser:**
```
Extract location from IP

Input: "ip": "8.8.8.8"
Output:
  "geo": {
    "country": "US",
    "city": "Mountain View",
    "lat": 37.386,
    "lon": -122.084
  }
```

**8. User-Agent Parser:**
```
Parse user agent string

Input: "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ..."
Output:
  "user_agent": {
    "os": "Windows 10",
    "browser": "Chrome",
    "device": "Desktop"
  }
```

### Example Pipeline

```
Pipeline: "Parse Python Application Logs"

Processors:
1. Grok Parser
   Pattern: %{date("yyyy-MM-dd HH:mm:ss"):timestamp} %{word:level} \[%{notSpace:logger}\] %{data:message}
   
2. Date Remapper
   Sources: timestamp
   
3. Status Remapper
   Sources: level
   
4. Category Processor
   Name: "Error categorization"
   Target: error.category
   Categories:
     - Filter: @error.type:DatabaseError
       Name: "database_error"
     - Filter: @error.type:TimeoutError
       Name: "timeout_error"
```

---

## 🔍 Searching Logs

### Search Syntax

**Basic Search:**
```
service:web-api
status:error
env:production
```

**Boolean Operators:**
```
service:web-api AND status:error
service:(web-api OR mobile-api)
service:web-api -status:info
```

**Wildcards:**
```
service:web-*
user.email:*@gmail.com
message:*timeout*
```

**Ranges:**
```
@http.response_time:[100 TO 500]
@user.age:>=18
@timestamp:[now-1h TO now]
```

**Exists:**
```
@error.type:*        # Has error.type field
-@user.id:*          # Doesn't have user.id
```

**Facets:**
```
@service:web-api @http.status_code:500
@user.tier:premium @country:US
```

### Search Examples

**Find Errors:**
```
status:error service:payment-api env:production
```

**Find Slow Requests:**
```
@http.response_time:>1000 service:api
```

**Find Specific User's Logs:**
```
@user.id:12345 -status:debug
```

**Find Logs Between Times:**
```
service:web-api @timestamp:[2026-01-01T10:00:00 TO 2026-01-01T11:00:00]
```

**Complex Query:**
```
service:payment-api 
AND status:(error OR warning) 
AND @transaction.amount:>1000 
AND @user.tier:premium 
AND @country:(US OR UK)
-@payment.method:test
```

---

## 🔗 Log Correlation

### Logs ↔ Traces

**Inject Trace Context:**

**Python:**
```python
import logging
from ddtrace import tracer

# Configure logging
logging.basicConfig(
    format='%(asctime)s %(levelname)s [dd.service=%(dd.service)s dd.env=%(dd.env)s dd.version=%(dd.version)s dd.trace_id=%(dd.trace_id)s dd.span_id=%(dd.span_id)s] - %(message)s'
)

logger = logging.getLogger(__name__)

# Datadog patches logging automatically
from ddtrace import patch
patch(logging=True)

# Now all logs include trace context
logger.info("Processing payment")  
# Output: 2026-01-01 10:30:15 INFO [dd.service=payment-api dd.env=production dd.version=v1.2.3 dd.trace_id=1234567890 dd.span_id=9876543210] - Processing payment
```

**JSON Format:**
```python
import json_logging
import logging

json_logging.init_non_web(enable_json=True)

logger = logging.getLogger(__name__)

# Automatically includes trace_id and span_id
logger.info("Payment processed", extra={
    'transaction_id': 'txn_123',
    'amount': 99.99
})

# Output:
# {
#   "message": "Payment processed",
#   "transaction_id": "txn_123",
#   "amount": 99.99,
#   "dd.trace_id": "1234567890",
#   "dd.span_id": "9876543210"
# }
```

**Node.js:**
```javascript
const tracer = require('dd-trace').init({
  logInjection: true
});

const winston = require('winston');

const logger = winston.createLogger({
  format: winston.format.json(),
  transports: [new winston.transports.Console()]
});

// Trace context automatically injected
logger.info('Payment processed', {
  transaction_id: 'txn_123',
  amount: 99.99
});
```

**In Datadog UI:**
```
1. Find trace in APM
2. Click "Logs" tab
3. See all logs from that trace

OR

1. Find log in Logs Explorer
2. Click log
3. See "View Trace" link
4. Click to see full trace
```

---

## 📊 Log-Based Metrics

### Create Metrics From Logs

**Use Case:**
```
Want to track error rate from logs
Without parsing every log manually
```

**Setup:**
```
1. Navigate to: Logs → Generate Metrics
2. Click "New Metric"
3. Configure:
   Name: error.count
   Filter: status:error service:payment-api
   Group by: @error.type, @user.tier
   
4. Metric created: error.count
   With tags: error_type, user_tier
   
5. Use in dashboards/monitors like any metric:
   sum:error.count{service:payment-api} by {error_type}
```

**Examples:**

**1. Error Rate by Service:**
```
Name: service.error_rate
Filter: status:error
Group by: service
Type: Count
```

**2. Slow Request Count:**
```
Name: slow_requests.count
Filter: @http.response_time:>1000
Group by: service, endpoint
Type: Count
```

**3. Login Attempts:**
```
Name: user.login.attempts
Filter: @event.type:login
Group by: @user.country, @auth.method
Type: Count
```

**4. Revenue from Logs:**
```
Name: transaction.revenue
Filter: @event.type:purchase
Measure: @transaction.amount
Group by: @country, @payment.method
Type: Distribution
```

---

## 🚨 Log Monitors

### Create Log-Based Alert

**Example 1: Error Rate Alert**
```
Monitor Type: Log Monitor

Query: status:error service:payment-api

Alert Condition:
  Above a threshold
  Alert threshold: 10 errors
  Warning threshold: 5 errors
  Evaluation window: last 5 minutes

Notify:
  @slack-alerts
  @pagerduty-payments
  
Message:
  {{#is_alert}}
  🚨 High error rate in payment-api
  {{value}} errors in last 5 minutes
  {{/is_alert}}
  
  Top errors:
  {{#logs}}
  - {{message}}
  {{/logs}}
  
  View logs: {{log.link}}
```

**Example 2: Specific Error Alert**
```
Query: 
  status:error 
  AND @error.type:DatabaseConnectionError 
  AND service:api

Alert when:
  Above 1 error in last 1 minute

Notify: @pagerduty-critical

Escalate: After 2 occurrences in 10 minutes
```

**Example 3: Anomaly Detection**
```
Query: status:error service:web-api

Alert Condition:
  Change Alert
  Compares: last 5 minutes vs same time yesterday
  Alert if: 50% increase

Useful for detecting unusual spikes
```

---

## 💰 Cost Optimization

### Indexing Strategy

**Pricing:**
```
Ingestion: $0.10 per GB
Indexing: $1.27 per million log events

Example:
100 GB ingested = $10
10% indexed (10M events) = $12.70
Total: $22.70
```

**Optimization:**

**1. Exclusion Filters:**
```
Exclude before indexing:

Filter 1: "Exclude Debug Logs"
  Query: status:debug
  → Don't index debug logs (save 60%)

Filter 2: "Exclude Health Checks"
  Query: @http.url:"/health"
  → Don't index health check logs

Filter 3: "Exclude Successful Background Jobs"
  Query: service:worker status:info @job.type:background
  → Only index errors from workers
```

**2. Sampling:**
```
Index only X% of logs:

Filter: "Sample Info Logs"
  Query: status:info
  Sample rate: 10%
  → Index only 10% of info logs
  → Still archive 100% to S3
```

**3. Index Only What You Alert On:**
```
✅ Index: Errors, warnings, business events
❌ Don't index: Debug, trace, verbose logs

Configure:
- status:(error OR warning)
- @event.type:(purchase OR signup OR payment)
```

**4. Shorter Retention:**
```
Default: 15 days
Consider: 3-7 days for most logs
Keep longer retention only for:
  - Security logs
  - Audit logs
  - Compliance logs
```

---

## 📝 Best Practices

### 1. Log Levels

```python
import logging

# Use appropriate levels
logging.debug("Variable value: %s", value)      # Development only
logging.info("User logged in: %s", user_id)     # Normal operations
logging.warning("Cache miss: %s", key)          # Something unusual
logging.error("Payment failed: %s", error)      # Requires attention
logging.critical("Database unreachable")        # System down
```

### 2. Structured Logging

```python
# ❌ Bad: Unstructured
logger.info(f"User {user_id} purchased product {product_id} for ${amount}")

# ✅ Good: Structured
logger.info("Purchase completed", extra={
    'event.type': 'purchase',
    'user.id': user_id,
    'product.id': product_id,
    'transaction.amount': amount,
    'transaction.currency': 'USD'
})
```

### 3. Context

```python
# Add context to every log
logger.info("Processing payment", extra={
    'user.id': user_id,
    'transaction.id': transaction_id,
    'payment.method': 'stripe',
    'amount': 99.99
})

# Later, can search: @transaction.id:txn_123
# See all logs for that transaction
```

### 4. Don't Log Sensitive Data

```python
# ❌ Bad
logger.info(f"Login: {username} / {password}")
logger.info(f"Credit card: {card_number}")

# ✅ Good
logger.info("Login successful", extra={
    'user.id': user_id,
    'ip': request.ip
})
logger.info("Payment processed", extra={
    'card.last4': card_last4,  # Only last 4 digits
    'card.brand': 'visa'
})
```

### 5. Log Sampling for High-Volume

```python
import random

# Sample 10% of info logs
if level == 'INFO' and random.random() < 0.1:
    logger.info(message)
    
# Always log errors
elif level in ['ERROR', 'CRITICAL']:
    logger.error(message)
```

---

## 📝 Tóm Tắt

### Key Concepts

```
1. COLLECTION
   - Agent file tailing
   - Container logs
   - HTTP API
   - Cloud provider integration

2. PROCESSING
   - Parsing (Grok)
   - Enrichment (GeoIP, User-Agent)
   - Filtering & Sampling

3. INDEXING
   - Index only what you need
   - Configure exclusion filters
   - Control costs

4. SEARCHING
   - Powerful query syntax
   - Faceted search
   - Boolean operators

5. CORRELATION
   - Logs ↔ Traces via trace_id
   - Logs ↔ Metrics
   - Unified troubleshooting
```

---

## 🎯 Bài Tập

### Exercise 1: Collect Logs

```
1. Enable logs in Agent
2. Configure log collection from file:
   /var/log/myapp/app.log
3. Add tags: env:lab, student:your-name
4. Generate logs (write to file)
5. View in Logs Explorer
```

### Exercise 2: Structured Logging

```python
# Write Python script that logs:

1. 5 INFO messages (JSON format)
2. 3 WARNING messages
3. 2 ERROR messages

Each log must include:
- timestamp
- level
- message
- user_id (random)
- transaction_id (uuid)
- amount (random 10-1000)

Send to Datadog via HTTP API
```

### Exercise 3: Log Search

```
In Logs Explorer:

1. Find all error logs from last hour
2. Find logs with transaction amount > 500
3. Find logs for specific user_id
4. Group by service
5. Create saved view
```

### Exercise 4: Log Monitor

```
Create monitor:

1. Alert when > 5 errors in 10 minutes
2. From your service
3. Send to email
4. Test by generating errors
5. Verify alert received
```

---

## ➡️ Bước Tiếp Theo

**Bài tiếp theo**: [09 - Infrastructure Monitoring](09-INFRASTRUCTURE.md)

---

**📌 Ghi Chú Của Bạn**
```








```

