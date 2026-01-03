# 12 - MONITORS VÀ ALERTS

## 🎯 Mục Tiêu Bài Học
Sau bài học này, bạn sẽ:
- Tạo monitors hiệu quả
- Configure alert conditions đúng
- Setup notification channels
- Prevent alert fatigue
- Implement on-call best practices

---

## 🚨 Monitors Overview

### Monitor Là Gì?

> **Monitor** = Automated rule kiểm tra metrics/logs/traces và gửi alert khi có vấn đề

**Ví dụ đơn giản:**
```
Giống như alarm đồng hồ:
- Set điều kiện: "Nếu CPU > 80%"
- Kiểm tra: Mỗi 1 phút
- Alert: Gửi Slack message khi trigger
```

### Monitor Lifecycle

```
┌─────────────────────────────────────┐
│     MONITOR STATES                  │
└─────────────────────────────────────┘

🟢 OK
  ↓ (Metric exceeds threshold)
🟡 WARN (Warning threshold crossed)
  ↓ (Metric continues to increase)
🔴 ALERT (Alert threshold crossed)
  ↓ (Send notifications)
📧 NOTIFIED (Team alerted)
  ↓ (Issue resolved)
✅ RESOLVED (Back to normal)
  ↓ (Send recovery notification)
🟢 OK
```

### Monitor Types

```
1. METRIC MONITOR
   → Alert on metric values

2. APM MONITOR  
   → Alert on trace metrics

3. LOG MONITOR
   → Alert on log patterns

4. COMPOSITE MONITOR
   → Combine multiple monitors

5. CHANGE MONITOR
   → Alert on metric changes

6. ANOMALY MONITOR
   → Detect unusual patterns

7. OUTLIER MONITOR
   → Find outliers in group

8. FORECAST MONITOR
   → Predict future issues

9. INTEGRATION MONITOR
   → External service status

10. CUSTOM CHECK
    → Custom agent checks
```

---

## 📊 Metric Monitor

### Basic Setup

**Create Monitor:**
```
1. Navigate to: Monitors → New Monitor → Metric

2. Define the metric:
   avg:system.cpu.user{env:production}

3. Set alert conditions:
   Alert threshold: > 80
   Warning threshold: > 60
   
4. Evaluation window:
   Average over last 5 minutes

5. Notify:
   @slack-alerts
   @email-team@company.com
   
6. Save
```

### Alert Conditions

**Threshold Alert:**
```yaml
Condition: Metric crosses a static value

Example:
  Metric: avg:system.cpu.user{*}
  Alert when: > 80%
  Warning when: > 60%
  
  ⚠️  Simple, but may cause false positives
```

**Change Alert:**
```yaml
Condition: Metric changes by X%

Example:
  Metric: avg:http.requests{*}
  Alert when: 
    - Increases > 50% compared to 1 hour ago
    OR
    - Decreases > 30% compared to 1 hour ago
    
  ✅  Good for detecting sudden traffic spikes/drops
```

**Anomaly Alert:**
```yaml
Condition: Metric behaves unusually

Example:
  Metric: avg:api.latency{*}
  Alert when: Outside bounds (2 standard deviations)
  
  Algorithm options:
    - Basic: Simple statistical bounds
    - Agile: Adapts quickly to changes  
    - Robust: Ignores outliers
    
  ✅  Best for metrics with patterns (daily, weekly)
```

**Forecast Alert:**
```yaml
Condition: Metric predicted to cross threshold

Example:
  Metric: avg:disk.used_pct{*}
  Alert when: Forecasted to reach 90% within 1 week
  
  ✅  Proactive, gives time to act
```

**Outlier Alert:**
```yaml
Condition: One member of group is different

Example:
  Metric: avg:system.cpu.user{*} by {host}
  Alert when: Any host is 2x higher than others
  
  ✅  Find problematic hosts/services
```

### Multi-Alert

**Alert on Each Group:**
```
Metric: avg:system.cpu.user{env:production} by {host}

Multi Alert: ON

Result:
- Creates separate alert for each host
- web-01: 85% → ALERT 🔴
- web-02: 45% → OK 🟢
- web-03: 65% → WARN 🟡

Notifications show which host has issue
```

**Simple Alert (No grouping):**
```
Multi Alert: OFF

Result:
- One alert for average across all hosts
- Average of all hosts: 65% → WARN 🟡

Less granular, may miss individual host issues
```

---

## ⚙️ Alert Configuration

### Evaluation Window

```
on average during the last X minutes
on average during the last X hours

Examples:

❌ Too short (1 minute):
   - Noisy, many false positives
   - React to temporary spikes
   
✅ Balanced (5 minutes):
   - Smooth out noise
   - Still responsive
   
❌ Too long (1 hour):
   - Slow to detect issues
   - Miss short incidents
   
Recommendation: 5-10 minutes for most metrics
```

### Advanced Options

**1. Require Full Window:**
```yaml
require_full_window: true

Behavior:
- Wait for full evaluation window before alerting
- Prevents alerts during startup/deployment
- Default: false

Use when:
✅ Deploying new instances
✅ Scaling up/down
❌ Need immediate alerts
```

**2. No Data:**
```yaml
notify_no_data: true
no_data_timeframe: 10 minutes

Behavior:
- Alert if metric stops reporting
- Important for critical services

Example:
- Payment service stops sending metrics → Alert!
- Agent crash → Alert!
```

**3. Auto Resolve:**
```yaml
notify_audit: false
locked: false

Behavior:
- Automatically resolve when metric returns to normal
- Send recovery notification

Most monitors should auto-resolve
```

**4. New Group Delay:**
```yaml
new_group_delay: 60 seconds

Behavior:
- Wait X seconds before evaluating new groups
- Prevents alerts during deployment of new hosts

Example:
- Deploy 10 new hosts
- Wait 60s for them to warm up
- Then start monitoring
```

**5. Evaluation Delay:**
```yaml
evaluation_delay: 60 seconds

Behavior:
- Wait X seconds for data to arrive
- Useful for delayed metrics (cloud integrations)

Example:
- AWS CloudWatch metrics have 5-15 min delay
- Set evaluation_delay: 900s (15 min)
```

---

## 📬 Notifications

### Notification Channels

**1. Email:**
```
@email-user@company.com
@email-team@company.com
```

**2. Slack:**
```
Setup:
1. Integrations → Slack → Add Account
2. Choose channel
3. In monitor: @slack-channel-name

Example: @slack-alerts
```

**3. PagerDuty:**
```
Setup:
1. Integrations → PagerDuty → Add Service
2. In monitor: @pagerduty-service-name

Features:
- On-call routing
- Escalation policies
- Incident management
```

**4. Webhooks:**
```
POST to custom URL when alert triggers

Use case:
- Send to internal systems
- Trigger automation
- Custom integrations
```

**5. Mobile Push:**
```
Datadog Mobile App
Get push notifications
```

### Notification Message

**Template:**
```
{{#is_alert}}
🚨 **ALERT**: {{name}}

**Value**: {{value}}
**Threshold**: {{threshold}}
**Host**: {{host.name}}

**What to check**:
1. Check dashboard: {{link}}
2. View logs: https://app.datadoghq.com/logs?query=service:{{service.name}}
3. Runbook: https://wiki.company.com/runbooks/high-cpu

{{/is_alert}}

{{#is_warning}}
⚠️  **WARNING**: {{name}}
Value: {{value}} (Threshold: {{warn_threshold}})
{{/is_warning}}

{{#is_recovery}}
✅ **RESOLVED**: {{name}}
Back to normal. Value: {{value}}
{{/is_recovery}}
```

**Available Variables:**
```
{{value}}              - Current metric value
{{threshold}}          - Alert threshold
{{warn_threshold}}     - Warning threshold
{{last_triggered_at}}  - When alert triggered

# Multi-alert variables
{{host.name}}
{{service.name}}
{{region}}
... any group-by tag

# Links
{{link}}               - Link to monitor
{{log.link}}           - Link to logs

# Conditionals
{{#is_alert}} ... {{/is_alert}}
{{#is_warning}} ... {{/is_warning}}
{{#is_recovery}} ... {{/is_recovery}}
{{#is_no_data}} ... {{/is_no_data}}
```

### Priority & Tagging

**Priority:**
```
P1 - Critical (Page on-call)
P2 - High (Slack + Email)
P3 - Medium (Email)
P4 - Low (Dashboard only)
```

**Tags:**
```
Monitor tags:
- team:backend
- service:payment-api
- severity:critical
- component:database

Use cases:
- Filter monitors by team
- Report on incidents by service
- Mute monitors by tag
```

---

## 🔕 Alert Fatigue Prevention

### Problem

```
Too many alerts → Team ignores them → Miss real issues

Causes:
❌ Threshold too sensitive
❌ Alert on everything
❌ No prioritization
❌ Noisy metrics
```

### Solutions

**1. Right Thresholds:**
```python
# Bad: Too sensitive
CPU > 50% → Alert

# Good: Based on actual impact
CPU > 80% for 5 minutes → Alert

# Better: Multiple levels
CPU > 70% → Warning (Email)
CPU > 85% → Alert (Slack)
CPU > 95% → Critical (PagerDuty)
```

**2. Alert on Symptoms, Not Causes:**
```
❌ Bad: "MySQL has 100 connections"
   → So what? Does it affect users?

✅ Good: "API response time > 1s"
   → User-facing impact, actionable
   
Then investigate: Oh, MySQL connections are high
```

**3. Composite Monitors:**
```
Alert only when MULTIPLE things wrong:

Monitor: "Critical Service Degradation"
Conditions:
  - (CPU > 80%) AND
  - (Response time > 1s) AND  
  - (Error rate > 1%)

Instead of 3 separate alerts → 1 actionable alert
```

**4. Maintenance Windows:**
```
During deployments/maintenance:
- Mute monitors temporarily
- Or use "Downtime" feature

Schedule downtime:
1. Monitors → Manage Downtime → Schedule
2. Select monitors or use tags
3. Set start/end time
4. Notify: Alerts muted during this time
```

**5. Alert Deduplication:**
```
If same alert triggers repeatedly:
- Don't send notification every time
- Use "Re-notify" settings

Example:
- First alert: Immediate
- If not resolved in 30 min: Re-notify
- Then every 1 hour until resolved
```

---

## 🎯 Monitor Examples

### Example 1: High CPU Alert

```yaml
Name: "{{host.name}} - High CPU Usage"
Type: Metric Monitor

Query: avg:system.cpu.user{env:production} by {host}

Conditions:
  Alert threshold: > 85
  Warning threshold: > 70
  Evaluation: last 5 minutes
  
Multi Alert: Yes (per host)

Notifications:
  Alert: 
    - @slack-infrastructure
    - @pagerduty-ops
  Warning:
    - @slack-infrastructure
    
Message: |
  {{#is_alert}}
  🚨 High CPU on {{host.name}}
  
  Current: {{value}}%
  Threshold: {{threshold}}%
  
  Actions:
  1. Check: https://app.datadoghq.com/dash/host/{{host.name}}
  2. SSH: ssh {{host.name}}
  3. Top processes: ps aux --sort=-%cpu | head
  4. Runbook: https://wiki/runbooks/high-cpu
  {{/is_alert}}

Tags:
  - team:ops
  - severity:high
  - component:infrastructure
```

### Example 2: API Error Rate

```yaml
Name: "{{service.name}} - High Error Rate"
Type: Metric Monitor

Query: 
  (sum:trace.http.request.errors{env:production} by {service}.as_count() / 
   sum:trace.http.request.hits{env:production} by {service}.as_count()) * 100

Conditions:
  Alert threshold: > 5  # 5% error rate
  Warning threshold: > 2
  Evaluation: last 10 minutes
  
Multi Alert: Yes (per service)

Notifications:
  Alert:
    - @pagerduty-engineering
    - @slack-alerts
  
Message: |
  🚨 High error rate in {{service.name}}
  
  Error rate: {{value}}%
  
  Quick checks:
  1. [View Traces](https://app.datadoghq.com/apm/traces?query=service:{{service.name}}%20status:error)
  2. [View Logs](https://app.datadoghq.com/logs?query=service:{{service.name}}%20status:error)
  3. [Service Dashboard](https://app.datadoghq.com/apm/service/{{service.name}})

Tags:
  - team:backend
  - severity:critical
```

### Example 3: Disk Space Forecast

```yaml
Name: "{{host.name}} - Disk Will Fill Up"
Type: Metric Monitor

Query: avg:system.disk.used{device:/dev/sda1} by {host}

Conditions:
  Type: Forecast
  Alert when: Predicted to reach 90% within 1 week
  Algorithm: linear
  
Multi Alert: Yes

Notifications:
  Alert: @slack-ops @email-ops
  
Message: |
  ⚠️  Disk space running out on {{host.name}}
  
  Current: {{value}}%
  Predicted: 90% in ~1 week
  
  Actions:
  1. Clean up logs: /var/log/*
  2. Check large files: du -h / | sort -rh | head -20
  3. Add disk space
  
Tags:
  - team:ops
  - severity:medium
  - proactive
```

### Example 4: Anomaly Detection

```yaml
Name: "Unusual Traffic Pattern"
Type: Metric Monitor

Query: avg:http.requests{env:production}.as_rate()

Conditions:
  Type: Anomaly
  Alert when: Above or Below bounds
  Algorithm: Agile
  Deviations: 2
  Evaluation: last 15 minutes

Notifications:
  Alert: @slack-security
  
Message: |
  🤖 Anomaly detected in traffic
  
  Current: {{value}} req/s
  Expected: {{lower_bound}} - {{upper_bound}} req/s
  
  Possible causes:
  - DDoS attack
  - Marketing campaign
  - Service outage
  - Bug causing retry storm
  
  Investigate: {{link}}
  
Tags:
  - team:security
  - type:anomaly
```

### Example 5: Composite Monitor

```yaml
Name: "Payment Service - Critical Degradation"
Type: Composite Monitor

Formula: (a && b) || c

Monitors:
  a: High latency (> 1s)
  b: High error rate (> 5%)
  c: Service down (no data)

Logic:
  Alert when:
    (Slow AND Errors) OR Down
  
  This prevents alert from:
    - Just slow but working
    - Few errors but fast
    
  Only alerts when real user impact

Notifications:
  Alert: 
    - @pagerduty-payments-critical
    - @slack-payments
    
Message: |
  🚨🚨 CRITICAL: Payment Service Degraded
  
  Conditions:
  {{#a}} - Latency: {{a.value}}s (> 1s) {{/a}}
  {{#b}} - Error rate: {{b.value}}% (> 5%) {{/b}}
  {{#c}} - Service: DOWN (no data) {{/c}}
  
  IMMEDIATE ACTIONS:
  1. Check service status
  2. Review recent deployments
  3. Check external dependencies (Stripe, DB)
  4. Escalate to L2 if not resolved in 15 min
  
Tags:
  - team:payments
  - severity:critical
  - impact:revenue
```

---

## 📊 Monitor Management

### Monitor Dashboard

```
View all monitors:
Monitors → Manage Monitors

Filters:
- Status: Triggered, OK, No Data
- Creator: user@company.com
- Tags: team:backend, severity:critical
- Muted: Yes/No

Bulk actions:
- Mute multiple monitors
- Edit tags
- Delete
```

### Monitor Groups

```
Organize monitors:

By Team:
- team:backend monitors
- team:frontend monitors
- team:ops monitors

By Service:
- service:payment-api monitors
- service:user-service monitors

By Severity:
- P1 monitors (Critical)
- P2 monitors (High)
```

### Monitor SLI Dashboard

```
Track monitor health:

Metrics:
- Total monitors
- Triggered monitors
- Muted monitors
- Average time to resolve
- Alert frequency

Dashboard shows:
- Which monitors alert most
- Which never alert (may need tuning)
- Resolution times by team
```

---

## 📝 Best Practices

### 1. Alert Naming

```
✅ Good:
"{{service.name}} - High Error Rate (>5%)"
"{{host.name}} - CPU Usage Critical"
"Payment Processing - Latency Exceeded"

Clear what's wrong and where

❌ Bad:
"Alert 1"
"Important Monitor"
"Check This"
```

### 2. Runbooks

```
Every alert should link to runbook:

Message should include:
1. What's wrong
2. Why it matters
3. How to investigate
4. How to fix
5. When to escalate

Example:
Runbook: https://wiki/runbooks/high-cpu
Contains:
- Common causes
- Investigation steps
- Resolution procedures
- Escalation path
```

### 3. On-Call Rotation

```
Setup:
1. Define on-call schedules (PagerDuty/Opsgenie)
2. P1 monitors → Page on-call
3. P2 monitors → Slack
4. P3 monitors → Email

Rotation:
- Primary on-call
- Secondary on-call (backup)
- Escalation after 15 min

Fair distribution of alerts across team
```

### 4. Post-Mortem

```
After each incident:
1. Root cause analysis
2. Timeline reconstruction
3. Action items

Monitor improvements:
- Adjust thresholds
- Add new monitors
- Remove noisy monitors
- Update runbooks
```

### 5. Monitor as Code

```yaml
# Terraform example
resource "datadog_monitor" "cpu" {
  name    = "High CPU Usage"
  type    = "metric alert"
  message = "CPU is high on {{host.name}}"
  
  query = "avg(last_5m):avg:system.cpu.user{env:production} by {host} > 85"
  
  thresholds = {
    critical = 85
    warning  = 70
  }
  
  notify_no_data = true
  no_data_timeframe = 10
  
  tags = ["team:ops", "severity:high"]
}
```

---

## 📝 Tóm Tắt

```
KEY POINTS:

1. MONITOR TYPES
   - Metric, APM, Log, Composite
   - Change, Anomaly, Forecast, Outlier

2. ALERT CONDITIONS
   - Threshold: Simple static value
   - Change: Compare to past
   - Anomaly: Detect unusual
   - Forecast: Predict future

3. NOTIFICATIONS
   - Multiple channels
   - Clear messages
   - Actionable information
   - Links to dashboards/logs

4. PREVENT ALERT FATIGUE
   - Right thresholds
   - Alert on symptoms
   - Composite monitors
   - Maintenance windows

5. BEST PRACTICES
   - Clear naming
   - Link to runbooks
   - Regular review
   - Monitor as code
```

---

## 🤔 Alerting Strategy Decisions

### Basic monitor design:

```
Create monitor:
1. Metric: system.cpu.user
2. Alert > 80%, Warn > 60%
3. Evaluation: 5 minutes
4. Multi-alert by host
5. Notify to your email
6. Generate load to trigger alert
7. Verify alert received
```

### Composite monitor approach:

```
Create composite monitor:
1. Monitor A: CPU > 80%
2. Monitor B: Memory > 80%
3. Composite: A AND B
4. Test: Trigger both
5. Verify single composite alert
```

### Anomaly detection configuration:

```
Create anomaly monitor:
1. Metric with daily pattern
2. Algorithm: Agile
3. Deviations: 2
4. Test by creating anomaly
5. Verify detection
```

### Monitor optimization:

```
Audit your monitors:
1. List all monitors
2. Find unused (never triggered)
3. Find noisy (trigger too often)
4. Adjust thresholds
5. Document changes
```

---

## ➡️ Bước Tiếp Theo

**Bài tiếp theo**: [16 - Best Practices](16-BEST-PRACTICES.md)

---

**📌 Ghi Chú Của Bạn**
```








```

