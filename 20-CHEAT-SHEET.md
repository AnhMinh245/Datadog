# 20 - DATADOG CHEAT SHEET

## 🚀 Quick Reference Guide

---

## 📦 Agent Commands

### Linux

```bash
# Status
sudo datadog-agent status

# Start/Stop/Restart
sudo systemctl start datadog-agent
sudo systemctl stop datadog-agent
sudo systemctl restart datadog-agent

# Enable auto-start
sudo systemctl enable datadog-agent

# Check specific integration
sudo datadog-agent check <CHECK_NAME>

# View logs
sudo tail -f /var/log/datadog/agent.log

# Configuration check
sudo datadog-agent config

# Flare (diagnostics bundle)
sudo datadog-agent flare <CASE_ID>
```

### Windows

```powershell
# Status
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" status

# Restart
Restart-Service datadogagent

# Check
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" check <CHECK_NAME>
```

### Docker

```bash
# Status
docker exec dd-agent agent status

# Logs
docker logs dd-agent -f

# Restart
docker restart dd-agent
```

---

## 📊 Query Language

### Metric Query Syntax

```
<aggregation>:<metric>{<filters>} [by {<tags>}]
```

**Aggregations:**
```
avg, sum, min, max, count
```

**Examples:**
```bash
# Average CPU across all hosts
avg:system.cpu.user{*}

# Sum requests by service
sum:http.requests{env:production} by {service}

# Max memory used by host
max:system.mem.used{*} by {host}

# Filter by multiple tags
avg:system.cpu.user{env:production,service:web-api}
```

### Functions

```bash
# Rate (convert count to per-second)
sum:http.requests{*}.as_rate()

# Count (convert rate to count)
sum:http.requests{*}.as_count()

# Rollup (custom time aggregation)
avg:system.cpu.user{*}.rollup(avg, 60)

# Timeshift (compare to past)
avg:system.cpu.user{*}
timeshift(avg:system.cpu.user{*}, 86400)  # -1 day

# Arithmetic
(avg:system.mem.total{*} - avg:system.mem.used{*}) / avg:system.mem.total{*} * 100

# Anomalies
anomalies(avg:system.cpu.user{*}, 'basic', 2)

# Forecast
forecast(avg:disk.used{*}, 'linear', 1w)

# EWMA (smoothing)
ewma_5(avg:system.cpu.user{*})
```

---

## 🔍 Log Search Syntax

### Basic Search

```bash
# By service
service:web-api

# By status
status:error

# By environment
env:production

# Combine
service:web-api status:error env:production
```

### Boolean Operators

```bash
# AND
service:web-api AND status:error

# OR
status:(error OR warning)
service:(web-api OR mobile-api)

# NOT
service:web-api -status:info
```

### Wildcards

```bash
# Prefix
service:web-*

# Suffix
user.email:*@gmail.com

# Contains
message:*timeout*
```

### Ranges

```bash
# Numeric range
@http.response_time:[100 TO 500]

# Greater than
@response_time:>1000

# Less than
@user.age:<18

# Time range
@timestamp:[now-1h TO now]
```

### Facets

```bash
# Has attribute
@user.id:*

# Doesn't have attribute
-@error.type:*

# Specific value
@http.status_code:500
@user.tier:premium
```

---

## 🏷️ Tags Reference

### Unified Service Tagging

```yaml
env:production
service:payment-api
version:v1.2.3
```

### Common Tags

```yaml
# Infrastructure
host:web-server-01
region:us-east-1
availability_zone:us-east-1a
instance_type:t3.large

# Application
service:payment-api
team:backend
component:api
language:python

# Environment
env:production
datacenter:us1

# Business
customer_tier:premium
feature:new_checkout
```

### Tag Rules

```
✅ Lowercase
✅ Use colons: key:value
✅ Low cardinality
✅ Descriptive

❌ No spaces
❌ No special characters
❌ High cardinality (user_id, request_id)
```

---

## 📨 Sending Metrics

### DogStatsD

```python
from datadog import statsd

# Gauge
statsd.gauge('my.metric', 100, tags=['env:prod'])

# Counter
statsd.increment('page.views', tags=['page:home'])
statsd.decrement('items.in_stock')
statsd.increment('sales.total', 5)  # Increment by 5

# Histogram
statsd.histogram('request.duration', 245)

# Distribution
statsd.distribution('file.size', 1024000)

# Timing (histogram for durations)
statsd.timing('database.query', 50)

# Set (unique counts)
statsd.set('unique.visitors', user_id)

# Context manager
with statsd.timed('my_function.duration'):
    my_function()
```

### HTTP API

```bash
curl -X POST "https://api.datadoghq.com/api/v1/series" \
  -H "Content-Type: application/json" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -d '{
    "series": [{
      "metric": "my.metric",
      "points": [['$(date +%s)', 100]],
      "type": "gauge",
      "tags": ["env:production"]
    }]
  }'
```

---

## 🔬 APM Instrumentation

### Python

```python
# Automatic
DD_SERVICE=my-app DD_ENV=prod DD_VERSION=1.0 ddtrace-run python app.py

# Manual
from ddtrace import tracer

@tracer.wrap(service='my-service', resource='process_order')
def process_order(order_id):
    pass

# Context manager
with tracer.trace('operation.name', service='my-service') as span:
    span.set_tag('user.id', user_id)
    do_work()
```

### Node.js

```javascript
// tracer.js
const tracer = require('dd-trace').init({
  service: 'my-app',
  env: 'production',
  version: '1.0.0'
});

// app.js
require('./tracer');  // Must be first!

// Custom span
const span = tracer.startSpan('operation.name');
span.setTag('user.id', userId);
span.finish();
```

### Java

```bash
java -javaagent:/path/to/dd-java-agent.jar \
     -Ddd.service=my-app \
     -Ddd.env=production \
     -Ddd.version=1.0.0 \
     -jar myapp.jar
```

---

## 📝 Log Configuration

### Agent Log Collection

```yaml
# /etc/datadog-agent/conf.d/app.d/conf.yaml
logs:
  - type: file
    path: /var/log/myapp/*.log
    service: my-app
    source: python
    tags:
      - env:production
```

### JSON Logging

```python
import logging
import json

class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_obj = {
            'timestamp': self.formatTime(record),
            'level': record.levelname,
            'message': record.getMessage(),
            'logger': record.name
        }
        return json.dumps(log_obj)

handler = logging.StreamHandler()
handler.setFormatter(JSONFormatter())
logger = logging.getLogger()
logger.addHandler(handler)
```

---

## 🚨 Monitor Quick Reference

### Monitor Types

```
Metric Monitor       - Alert on metric values
APM Monitor          - Alert on traces/spans
Log Monitor          - Alert on log patterns
Composite Monitor    - Combine multiple monitors
Change Monitor       - Alert on metric changes
Anomaly Monitor      - Detect unusual patterns
Forecast Monitor     - Predict future issues
Outlier Monitor      - Find anomalies in groups
```

### Alert Conditions

```yaml
# Threshold
avg:system.cpu.user{*} > 80

# Change
avg:http.requests{*} increases > 50% compared to 1h ago

# Anomaly
anomalies(avg:metric{*}, 'basic', 2)

# Forecast
forecast(avg:disk.used{*}, 'linear', 1w) >= 90
```

### Notification Variables

```
{{value}}              - Current value
{{threshold}}          - Alert threshold
{{warn_threshold}}     - Warning threshold
{{host.name}}          - Host name
{{service.name}}       - Service name
{{link}}               - Link to monitor
{{log.link}}           - Link to logs

# Conditionals
{{#is_alert}} ... {{/is_alert}}
{{#is_warning}} ... {{/is_warning}}
{{#is_recovery}} ... {{/is_recovery}}
```

---

## 🔧 Configuration Files

### Agent Main Config

```yaml
# /etc/datadog-agent/datadog.yaml

api_key: <YOUR_API_KEY>
site: datadoghq.com
hostname: my-server

tags:
  - env:production
  - team:backend

logs_enabled: true

apm_config:
  enabled: true
  apm_non_local_traffic: true

process_config:
  enabled: "true"

dogstatsd_port: 8125
```

### Integration Config

```yaml
# /etc/datadog-agent/conf.d/postgres.d/conf.yaml
init_config:

instances:
  - host: localhost
    port: 5432
    username: datadog
    password: <PASSWORD>
    tags:
      - env:production
      - role:primary
```

---

## 🐳 Docker & Kubernetes

### Docker Compose

```yaml
version: '3'
services:
  datadog:
    image: datadog/agent:latest
    environment:
      - DD_API_KEY=${DD_API_KEY}
      - DD_SITE=datadoghq.com
      - DD_LOGS_ENABLED=true
      - DD_LOGS_CONFIG_CONTAINER_COLLECT_ALL=true
      - DD_APM_ENABLED=true
      - DD_APM_NON_LOCAL_TRAFFIC=true
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /proc/:/host/proc/:ro
      - /sys/fs/cgroup/:/host/sys/fs/cgroup:ro
    ports:
      - "8126:8126"
      - "8125:8125/udp"
```

### Kubernetes Helm

```bash
helm repo add datadog https://helm.datadoghq.com
helm install datadog-agent datadog/datadog \
  --set datadog.apiKey=<API_KEY> \
  --set datadog.site=datadoghq.com \
  --set datadog.logs.enabled=true \
  --set datadog.apm.portEnabled=true
```

---

## 💰 Cost Optimization

### Reduce Metric Cardinality

```python
# ❌ Bad: 1000 custom metrics
for i in range(1000):
    statsd.gauge(f'metric.{i}', value)

# ✅ Good: 1 custom metric
statsd.gauge('metric.aggregate', sum(values))
```

### Log Exclusion Filters

```bash
# Exclude health checks
@http.url:"/health"

# Exclude debug logs
status:debug

# Sample info logs (10%)
status:info
```

### APM Sampling

```yaml
apm_config:
  analyzed_spans:
    service-name|*: 0.1  # 10%
    service-name|error: 1.0  # 100% errors
```

---

## 🔗 Useful URLs

### Datadog Sites

```
US1:  https://app.datadoghq.com
US3:  https://us3.datadoghq.com
US5:  https://us5.datadoghq.com
EU1:  https://app.datadoghq.eu
```

### API Endpoints (US1)

```
Metrics:  https://api.datadoghq.com/api/v1/series
Logs:     https://http-intake.logs.datadoghq.com/v1/input
Traces:   https://trace.agent.datadoghq.com
```

### Documentation

```
Main Docs:     https://docs.datadoghq.com/
API Docs:      https://docs.datadoghq.com/api/
Learning:      https://learn.datadoghq.com/
Status:        https://status.datadoghq.com/
```

---

## 🎯 Common Use Cases

### High CPU Alert

```yaml
Monitor: Metric Alert
Query: avg:system.cpu.user{*} by {host} > 85
Evaluation: last 5 minutes
Multi-alert: Yes
Notify: @pagerduty @slack-ops
```

### Error Rate Alert

```yaml
Monitor: Metric Alert  
Query: (sum:trace.http.request.errors{*} / sum:trace.http.request.hits{*}) * 100 > 5
Evaluation: last 10 minutes
Notify: @pagerduty-engineering
```

### Log Pattern Alert

```bash
Monitor: Log Alert
Query: status:error service:payment-api
Threshold: > 10 errors in 5 minutes
Notify: @slack-alerts
```

### Disk Space Forecast

```yaml
Monitor: Forecast Monitor
Query: avg:system.disk.used_pct{*} by {host}
Alert when: Predicted to reach 90% in 1 week
Notify: @email-ops
```

---

## 🔑 Environment Variables

### Agent

```bash
DD_API_KEY=<your_key>
DD_SITE=datadoghq.com
DD_HOSTNAME=my-host
DD_TAGS="env:prod service:api"
DD_LOGS_ENABLED=true
DD_APM_ENABLED=true
DD_PROCESS_AGENT_ENABLED=true
```

### APM

```bash
DD_SERVICE=my-service
DD_ENV=production
DD_VERSION=v1.2.3
DD_TRACE_SAMPLE_RATE=1.0
DD_TRACE_AGENT_HOSTNAME=localhost
DD_TRACE_AGENT_PORT=8126
```

---

## 🛠️ Troubleshooting

### Agent Not Sending Data

```bash
# 1. Check agent status
sudo datadog-agent status

# 2. Check connectivity
curl -v https://api.datadoghq.com

# 3. Check logs
sudo tail -f /var/log/datadog/agent.log

# 4. Verify API key
sudo datadog-agent config | grep api_key

# 5. Run diagnostics
sudo datadog-agent diagnose
```

### High CPU/Memory Usage

```yaml
# Reduce check frequency
min_collection_interval: 30

# Reduce log file limit
logs_config:
  open_files_limit: 50

# Disable unused features
process_config:
  enabled: "false"
```

### Missing Metrics

```bash
# Check integration
sudo datadog-agent check <integration>

# Check config syntax
sudo datadog-agent configcheck

# Force flush
sudo datadog-agent flare
```

---

## 📝 Quick Tips

```
✅ Always use unified service tagging (env, service, version)
✅ Keep tag cardinality low (< 1000 unique values)
✅ Use structured JSON logging
✅ Alert on symptoms, not causes
✅ Include runbooks in alert messages
✅ Sample high-volume metrics/logs
✅ Use template variables in dashboards
✅ Regular cost optimization reviews
✅ Infrastructure as Code for monitors
✅ Correlate metrics, logs, and traces
```

---

## 📚 Learning Path

```
1. Setup Agent ✅
2. Create Dashboard ✅
3. Send Custom Metrics ✅
4. Setup APM ✅
5. Configure Logs ✅
6. Create Monitors ✅
7. Define SLOs
8. Optimize Costs
9. Infrastructure as Code
10. Advanced Features
```

---

**🎉 Chúc mừng! Bạn đã hoàn thành khóa học Datadog!**

Keep this cheat sheet handy for quick reference. Practice makes perfect!

---

**📌 Personal Notes**
```
(Add your own shortcuts, commands, and tips here)








```

