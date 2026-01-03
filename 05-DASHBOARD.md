# 05 - DASHBOARD STRATEGY & DESIGN

## 🎯 Mục Tiêu
Hiểu dashboard design principles, widget selection, query optimization và best practices để tạo dashboards hiệu quả cho stakeholders và teams.

---

## 📚 Bản Chất: Dashboards Là Gì?

### **Purpose của Dashboards**

```
Dashboard ≠ Dump all metrics
Dashboard = Tell a story with data

3 loại stories:

1. Operational Story
   "Is the system healthy RIGHT NOW?"
   → For: On-call engineers
   → Real-time status, quick triage
   
2. Analytical Story
   "WHY did this happen?"
   → For: Performance engineers
   → Deep dive, correlation, trends
   
3. Business Story
   "What's the IMPACT?"
   → For: Executives, product managers
   → High-level KPIs, SLOs, business metrics
```

**Common Mistake:**
```
❌ One dashboard for everyone
   → Too complex for execs
   → Not detailed enough for engineers
   
✅ Purpose-built dashboards
   → Executive dashboard (10 widgets, high-level)
   → Engineering dashboard (50 widgets, detailed)
   → On-call dashboard (20 widgets, actionable)
```

---

## 🎨 Dashboard Types: When to Use What

### **Type 1: Timeboard**

**Definition:**
> Shared timeline across all widgets - zoom/pan affects everything

```
┌─────────────────────────────────────────────┐
│ [====== Shared Timeline: Last 4 Hours =====]│
├─────────────────────────────────────────────┤
│  CPU Usage          │  Memory Usage        │
│  [graph synced]     │  [graph synced]      │
├─────────────────────────────────────────────┤
│  Request Rate       │  Error Rate          │
│  [graph synced]     │  [graph synced]      │
└─────────────────────────────────────────────┘
```

**When to Use:**
```
✅ Troubleshooting / Investigation
✅ Correlation analysis
✅ Incident response
✅ Performance debugging

Example: "Response time spiked at 10 AM"
→ Timeboard shows CPU, memory, requests at 10 AM across all widgets
→ Easy to correlate
```

**Pros:**
- ✅ Zoom timeline → all widgets adjust
- ✅ See correlations easily
- ✅ Great for "what happened when"

**Cons:**
- ⚠️ All widgets must use same timeframe
- ⚠️ Not flexible for mixed use cases

---

### **Type 2: Screenboard**

**Definition:**
> Independent timeline per widget - flexible layout

```
┌─────────────────────────────────────────────┐
│  Total Users (Last 24h)  │  SLO (Last 30d) │
│  [independent time]      │  [independent]  │
├─────────────────────────────────────────────┤
│  Recent Deployments           │  Text Note │
│  (Last 7 days)                │            │
└─────────────────────────────────────────────┘
```

**When to Use:**
```
✅ Executive dashboards
✅ Status boards / NOC displays
✅ Mixed timeframes needed
✅ Public dashboards

Example: Executive Dashboard
→ SLO widget: Last 30 days
→ Revenue widget: Last 24 hours
→ Deploy widget: Last 7 days
→ Different timeframes make sense
```

**Pros:**
- ✅ Flexible layout (drag-and-drop)
- ✅ Each widget independent
- ✅ Can embed images, text, iframes

**Cons:**
- ⚠️ No shared timeline (harder to correlate)

---

### **Decision Framework:**

```
Need to correlate events?
├─ YES → Timeboard
│   Examples:
│   - Troubleshooting dashboard
│   - Performance analysis
│   - Incident investigation
│
└─ NO → Screenboard
    Examples:
    - Executive overview
    - Team status board
    - Mixed metrics (different timeframes)
```

---

## 🧩 Widget Selection Guide

### **Decision Tree: Which Widget?**

```
What are you showing?

├─ Single number (current value)
│  └─ Use: Query Value
│     Examples: Current CPU %, Active users, Error count
│
├─ Change over time
│  └─ Use: Timeseries
│     Examples: Request rate, Response time trend, Memory usage
│
├─ Ranking (top/bottom N)
│  └─ Use: Top List
│     Examples: Top hosts by CPU, Slowest endpoints, Most errors
│
├─ Distribution of values
│  ├─ Histogram → Use: Distribution
│  │  Examples: Latency percentiles (p50/p95/p99)
│  │
│  └─ Heatmap → Use: Heatmap
│     Examples: Request duration across services, Error patterns
│
├─ Real-time events
│  └─ Use: Log Stream
│     Examples: Live errors, Deployment logs, Audit trail
│
├─ Infrastructure overview
│  └─ Use: Host Map
│     Examples: Fleet health, Resource utilization across hosts
│
├─ Service relationships
│  └─ Use: Service Map
│     Examples: Microservice dependencies, Data flow
│
└─ Multiple metrics comparison
   └─ Use: Table
      Examples: Host comparison (CPU, RAM, Disk), Service comparison
```

---

### **Widget Deep Dive**

#### **Query Value - Single Number**

**Purpose:** Show current state or aggregate value

**Use Cases:**
```
✅ Current CPU usage: 67.5%
✅ Total requests today: 1.2M
✅ Active users: 850
✅ Error rate: 0.3%
```

**Design Considerations:**
```yaml
Conditional Formatting (Critical):
  < 50%: Green (good)
  50-80%: Yellow (warning)
  > 80%: Red (critical)
  
Unit Selection:
  - Percentage: %
  - Duration: ms, s
  - Throughput: /s, /min
  - Count: format with commas (1,234,567)

Font Size:
  - Large for dashboards on TVs
  - Smaller for dense dashboards
```

**Common Mistake:**
```
❌ Query Value for trend data
   Example: response_time query value = 250ms
   → Missing: Is it getting better or worse?
   
✅ Use Query Value + Timeseries
   → Value shows current: 250ms
   → Timeseries shows trend: increasing!
```

---

#### **Timeseries - Line/Area/Bar Chart**

**Purpose:** Show changes over time

**Visualization Options:**
```
Lines:
  ✅ Multiple series comparison
  ✅ Sparse data
  Example: Compare CPU across 5 hosts

Area:
  ✅ Show cumulative effect
  ✅ Emphasize magnitude
  Example: Total requests (stacked by service)

Bars:
  ✅ Discrete events
  ✅ Clear separation
  Example: Deployments per day, Errors per hour
```

**Display Modes:**
```yaml
Lines:
  - Interpolation: linear, step, cardinal
  - Line width: thin (multiple series), thick (single series)
  
Bands (with lines):
  - Show bounds/uncertainty
  - Example: Anomaly detection bounds
  
Both:
  - Combination for clarity
```

**Y-Axis Decisions:**
```
Auto-scale (default):
  ✅ Good for exploration
  ⚠️ Can be misleading (exaggerates changes)
  
Fixed scale:
  ✅ Consistent interpretation
  ✅ Compare across dashboards
  Example: CPU always 0-100%, not auto-scale 60-80%
  
Log scale:
  ✅ When range varies greatly
  Example: Error counts (1 to 10,000)
```

---

#### **Top List - Rankings**

**Purpose:** Show top/bottom N items

**Use Cases:**
```
✅ Top 10 hosts by CPU
✅ Slowest 5 endpoints
✅ Most active users (careful: cardinality!)
✅ Largest database tables
```

**Configuration:**
```yaml
Order:
  Descending: Top N highest
  Ascending: Bottom N (lowest)
  
Limit:
  Default: 10
  Consideration: More = harder to read
  
Conditional Format:
  Highlight top 1 in red (if bad metric)
  Highlight top 1 in green (if good metric)
```

**Gotcha:**
```
❌ Top list by high-cardinality tag
   Example: Top 100 users by requests
   → Creates 100 custom metrics
   → Expensive!
   
✅ Top list by low-cardinality tag
   Example: Top 10 services by requests
   → Max 10 metrics
```

---

#### **Heatmap - Pattern Visualization**

**Purpose:** Show distribution across dimensions

**Use Cases:**
```
✅ Response time distribution across services
✅ Error patterns by hour of day
✅ Request volume by region and time
```

**When to Use:**
```
✅ When you have 2 dimensions
   Example: Service (X) × Time (Y)
   
✅ When you want to see patterns
   Example: "Service A always slow at midnight"
   
✅ When exact numbers less important than trends
```

**Color Schemes:**
```
Sequential (default):
  - Light → Dark (low → high)
  - Good for most use cases
  
Diverging:
  - Blue ← Neutral → Red
  - Good for deltas (negative/positive)
```

---

## 🔍 Query Language Mastery

### **Query Anatomy**

```
<aggregation>:<metric>{<filters>} [by {<tags>}] [functions]
     ▲           ▲         ▲            ▲           ▲
     │           │         │            │           │
   What to   Which    Which       Group by     Transform
     do      metric   subset       what
```

### **Aggregations: What To Do**

```
avg:   Average (most common)
sum:   Total (for counts, requests)
min:   Minimum value
max:   Maximum value (peak)
count: Number of data points
```

**When to use:**
```yaml
avg:
  - CPU usage (average across time)
  - Response time (mean)
  - Memory usage
  
sum:
  - Request counts (total requests)
  - Error counts (total errors)
  - Bytes transferred (total bandwidth)
  
max:
  - Peak CPU (worst case)
  - Max response time (slowest request)
  - Highest queue depth
  
min:
  - Available disk space (lowest point)
  - Minimum connections (baseline)
```

---

### **Filters: Which Subset**

```bash
# Single tag
{env:production}

# Multiple tags (AND)
{env:production,service:api}

# OR logic
{env:production OR env:staging}
{service:api OR service:worker}

# Wildcards
{service:web-*}
{host:prod-db-*}

# NOT
{env:production,-service:test}
```

**Best Practice:**
```yaml
Specific is better:
❌ {*}  # All data → expensive, slow
✅ {env:production,service:payment-api}  # Precise

Use template variables:
❌ Hardcoded: {env:production}
✅ Dynamic: {env:$env}
   → Users can switch env via dropdown
```

---

### **Group By: Split Data**

```bash
# Single dimension
avg:system.cpu.user{*} by {host}
→ One line per host

# Multiple dimensions
sum:http.requests{*} by {service,status_code}
→ One line per service+status combo
```

**Cardinality Warning:**
```
Group by creates: N × M series

Example:
- 10 services
- 5 status codes (200, 400, 404, 500, 503)
= 50 series in one graph

Too many series → unreadable graph
Limit: Keep under 20-30 series per graph
```

---

### **Functions: Transform Data**

#### **as_rate() / as_count()**

```bash
# Convert count to rate (per second)
sum:http.requests{*}.as_rate()
→ Requests per second

# Convert rate to count
sum:http.requests{*}.as_count()
→ Total requests in timeframe
```

**When to use:**
```
as_rate():
  ✅ Counts that accumulate (requests, errors)
  ✅ Want per-second rate for comparison
  
as_count():
  ✅ See total over period
  Example: Total errors in last hour
```

---

#### **rollup() - Time Aggregation**

```bash
# Average over custom window
avg:system.cpu.user{*}.rollup(avg, 60)
→ Average CPU in 60-second windows

# Sum over window
sum:http.requests{*}.rollup(sum, 300)
→ Total requests per 5 minutes
```

**Use Case:**
```
Long timeframes (days/weeks):
→ Default rollup = 1 point per pixel
→ Can miss spikes

Custom rollup:
→ rollup(max, 60)
→ Shows peak value in each window
→ Won't miss spikes
```

---

#### **timeshift() - Compare to Past**

```bash
# Compare to yesterday
avg:system.cpu.user{*}
timeshift(avg:system.cpu.user{*}, 86400)

# Compare to last week
timeshift(avg:system.cpu.user{*}, 604800)
```

**Seconds conversion:**
```
1 hour   = 3600
1 day    = 86400
1 week   = 604800
1 month  = 2592000 (30 days)
```

**Use Case:**
```
Banking example:
- Compare transaction volume today vs last Monday
- Identify weekly patterns
- Validate capacity planning
```

---

#### **anomalies() - ML Detection**

```bash
anomalies(avg:system.cpu.user{*}, 'basic', 2)
           │                        │       │
           metric                algorithm bounds
```

**Algorithms:**
```
basic:
  - Fast, simple
  - Good for stable metrics
  
agile:
  - Adapts quickly to changes
  - Good for dynamic metrics
  
robust:
  - Ignores outliers
  - Good for noisy data
```

**Bounds:**
```
2 = 2 standard deviations
  → Moderate sensitivity
  
3 = 3 standard deviations
  → Lower sensitivity (fewer false positives)
  
1 = 1 standard deviation
  → High sensitivity (more alerts)
```

---

#### **forecast() - Predict Future**

```bash
forecast(avg:system.disk.used{*}, 'linear', 1w)
         │                         │         │
         metric                  algorithm  horizon
```

**Use Case:**
```
Capacity planning:
- When will disk be full?
- When to add more servers?
- Budget planning
```

**Banking Example:**
```
Transaction growth:
forecast(sum:transactions{*}.as_rate(), 'linear', 1mo)

Shows: "At current rate, will hit 10K TPS in 3 weeks"
Action: Plan infrastructure upgrade
```

---

## 🎯 Dashboard Design Patterns

### **Pattern 1: RED Method Dashboard**

**For: Service/API monitoring**

```
RED = Rate, Errors, Duration

┌─────────────────────────────────────────────┐
│  Service: Payment API                        │
├─────────────────────────────────────────────┤
│  Rate (Traffic)                              │
│  - Requests per second                       │
│  - Timeseries: sum:requests{*}.as_rate()    │
├─────────────────────────────────────────────┤
│  Errors (Error Rate)                         │
│  - Error percentage                          │
│  - Query: (errors / total) * 100            │
├─────────────────────────────────────────────┤
│  Duration (Latency)                          │
│  - p50, p95, p99 response time              │
│  - Distribution widget                       │
└─────────────────────────────────────────────┘
```

**Why RED:**
- ✅ Complete service health picture
- ✅ Industry standard
- ✅ Easy to understand

---

### **Pattern 2: USE Method Dashboard**

**For: Infrastructure monitoring**

```
USE = Utilization, Saturation, Errors

┌─────────────────────────────────────────────┐
│  Resource: CPU                               │
├─────────────────────────────────────────────┤
│  Utilization                                 │
│  - % of CPU used                             │
│  - avg:system.cpu.user{*}                   │
├─────────────────────────────────────────────┤
│  Saturation                                  │
│  - Load average, queue depth                 │
│  - avg:system.load.1{*}                     │
├─────────────────────────────────────────────┤
│  Errors                                      │
│  - CPU throttling, context switches          │
└─────────────────────────────────────────────┘
```

**Apply to:**
- CPU, Memory, Disk, Network
- Database connections
- Queue systems

---

### **Pattern 3: Golden Signals (Google SRE)**

```
4 Golden Signals: Latency, Traffic, Errors, Saturation

Similar to RED + USE, but adds:
- Saturation (resource fullness)

Banking implementation:
┌─────────────────────────────────────────────┐
│  Latency: p95 transaction time < 200ms      │
│  Traffic: 1000 TPS current                  │
│  Errors: 0.1% error rate                    │
│  Saturation: Database connections 70% used  │
└─────────────────────────────────────────────┘
```

---

### **Pattern 4: Hierarchical (Top-Down)**

```
Executive View → Team View → Service View → Resource View

Example: Banking Dashboard Hierarchy

Level 1 (Executive):
├─ Overall SLO: 99.95%
├─ Total Transactions: 1.2M today
├─ Revenue: $500K today
└─ Critical Incidents: 0

Level 2 (Team Lead):
├─ Service Status (all services)
├─ Team-specific SLOs
├─ Recent Deployments
└─ Top Issues

Level 3 (Engineer):
├─ Detailed service metrics
├─ Error breakdown
├─ Performance deep dive
└─ Resource utilization

Level 4 (Deep Dive):
├─ Individual host metrics
├─ Database queries
├─ Log streams
└─ Trace analysis
```

---

## 🎨 Design Best Practices

### **1. Layout & Organization**

**Grid System:**
```
✅ Use consistent widget sizes
   - Full width for titles
   - 50% width for paired metrics
   - 33% width for trios
   
❌ Random sizes and placement
   - Hard to scan
   - Looks unprofessional
```

**Logical Flow:**
```
Top to Bottom priority:
1. Most important metrics (top)
2. Supporting context (middle)
3. Deep dive / details (bottom)

Left to Right:
1. Input metrics (left)
2. Processing metrics (middle)
3. Output metrics (right)

Example: API Dashboard
┌──────────────┬──────────────┬──────────────┐
│ Request Rate │ Process Time │ Response Rate│
│ (Input)      │ (Middle)     │ (Output)     │
└──────────────┴──────────────┴──────────────┘
```

---

### **2. Color Usage**

**Standard Colors:**
```
Green: Good / Normal / Success
Yellow: Warning / Degraded
Red: Critical / Error / Alert
Blue: Info / Neutral
Purple: Custom / Secondary metric
```

**Consistency:**
```
✅ Same colors for same concepts across all dashboards
   - Errors always red
   - Success always green
   
❌ Random colors
   - Errors sometimes red, sometimes orange
   - Confusing for teams
```

**Color Blindness:**
```
⚠️ 8% of men are color blind (red-green most common)

Accessible approach:
✅ Use shapes + colors
✅ Use text labels
✅ Use patterns (solid, dashed, dotted)
```

---

### **3. Naming Conventions**

**Dashboard Names:**
```
Pattern: [Team/System] - [Purpose]

Examples:
✅ "Payment API - Production Health"
✅ "Core Banking - Executive Overview"
✅ "Infrastructure - On-Call Dashboard"

❌ "Dashboard 1"
❌ "John's Dashboard"
❌ "Temp Dashboard"
```

**Widget Titles:**
```
Pattern: [Metric Name] - [Context]

Examples:
✅ "Response Time - p95 (Last 1 hour)"
✅ "Error Rate % - By Service"
✅ "Database Connections - PostgreSQL Primary"

❌ "Graph 1"
❌ "Metric"
❌ "system.cpu.user"
```

---

### **4. Context & Documentation**

**Add Notes Widgets:**
```
Purpose:
- Explain what metrics mean
- Document thresholds
- Link to runbooks
- Provide business context

Example:
┌────────────────────────────────────────────┐
│ 📝 Response Time Targets                   │
│                                             │
│ Target: p95 < 200ms                        │
│ Warning: p95 > 200ms                       │
│ Critical: p95 > 500ms                      │
│                                             │
│ Runbook: wiki.company.com/api-slow         │
│ On-call: Slack #team-backend              │
└────────────────────────────────────────────┘
```

**Use Markdown:**
```markdown
## Response Time SLO

**Target**: p95 < 200ms  
**Current**: 185ms ✅

### Actions if breached:
1. Check database slow query log
2. Review recent deployments
3. Escalate to @backend-lead

[Runbook](https://wiki.company.com/slo-breach)
```

---

### **5. Template Variables**

**Purpose:**
```
Make dashboards reusable across:
- Environments (prod, staging, dev)
- Services (api, web, worker)
- Teams (backend, frontend, data)
```

**Implementation:**
```yaml
Variable 1: $env
  Tag: env
  Default: production
  Values: production, staging, development

Variable 2: $service
  Tag: service
  Default: *
  Values: (auto from tag)

Variable 3: $region
  Tag: region
  Default: *
  Values: us-east-1, us-west-2, eu-west-1

Usage in queries:
avg:system.cpu.user{env:$env,service:$service,region:$region}
```

**User Experience:**
```
Before variables:
- Need separate dashboard per env = 3 dashboards
- Change service = edit every query

With variables:
- 1 dashboard for all envs
- Dropdown to switch env/service
- No editing needed
```

---

## ⚠️ Common Mistakes

### **Mistake 1: Too Many Widgets**

```
❌ Problem: 100 widgets on one dashboard
   → Information overload
   → Slow load time
   → Nothing stands out
   
✅ Solution: Multiple focused dashboards
   Dashboard 1: Overview (10 widgets)
   Dashboard 2: Deep Dive CPU (20 widgets)
   Dashboard 3: Deep Dive Network (20 widgets)
```

---

### **Mistake 2: No Aggregation**

```
❌ Problem: 
   Query: avg:system.cpu.user{*} by {host}
   Result: 1000 lines (one per host)
   → Unreadable

✅ Solution: Aggregate appropriately
   Option 1: avg across all hosts
     avg:system.cpu.user{*}
   
   Option 2: Top 10 only
     Top List widget
     
   Option 3: Percentiles
     p95:system.cpu.user{*}
```

---

### **Mistake 3: Wrong Timeframe**

```
❌ CPU dashboard showing last 5 years
   → Can't see recent spikes
   
✅ Match timeframe to use case:
   Real-time monitoring: Last 1 hour
   Daily operations: Last 4 hours
   Trends: Last 7 days
   Capacity planning: Last 30 days
```

---

### **Mistake 4: No Thresholds**

```
❌ Just show metrics
   → Users don't know if good or bad
   
✅ Add context:
   - Horizontal line at threshold
   - Conditional formatting
   - Notes with targets
   
Example: CPU graph
- Add red line at 80% (critical)
- Add yellow line at 60% (warning)
- Note: "Target: < 60% average"
```

---

## 📊 Banking Dashboard Examples

### **Executive Dashboard**

```yaml
Purpose: High-level business + tech metrics
Audience: CTO, VP Engineering, Business Leaders
Refresh: 5 minutes
Type: Screenboard

Widgets (12 total):

Row 1 (KPIs):
  - Total Transactions Today (Query Value)
  - Transaction Success Rate % (Query Value, conditional format)
  - Average Response Time (Query Value)
  - SLO Status - 99.9% (SLO Widget)

Row 2 (Trends):
  - Transactions Per Hour (Timeseries)
  - Revenue Per Hour (Timeseries, business metric)
  
Row 3 (Ops):
  - Active Incidents (Query Value)
  - Recent Deployments (Event Stream)
  - System Health (Host Map)
  
Row 4 (Notes):
  - Status Message (Text: "All systems operational ✅")
  - On-Call Contact (Text: current on-call engineer)
```

---

### **On-Call Dashboard**

```yaml
Purpose: Quick triage and incident response
Audience: On-call engineer
Refresh: 1 minute (auto)
Type: Timeboard (shared timeline for correlation)

Widgets (20 total):

Critical Alerts (Top):
  - Active Alerts (Monitor Status Widget)
  - Error Rate % (Query Value, big, red if > 1%)
  
Service Health:
  - Request Rate (Timeseries)
  - Response Time p95/p99 (Timeseries)
  - Error Count (Timeseries)
  - Top Errors by Endpoint (Top List)
  
Infrastructure:
  - CPU Usage by Host (Timeseries)
  - Memory Usage by Host (Timeseries)
  - Disk Usage (Host Map)
  
Logs:
  - Error Logs (Log Stream, live)
  - Recent Deployments (Event Timeline)
  
Context:
  - Runbook Links (Text widget)
  - Escalation Contacts (Text widget)
```

---

### **Service Deep Dive**

```yaml
Service: Payment Processing API
Purpose: Detailed performance analysis
Audience: Payment team engineers
Type: Timeboard

Widgets:

RED Metrics:
  - Request Rate (total, by endpoint)
  - Error Rate (total, by error type)
  - Latency Distribution (p50/p75/p95/p99)
  
Dependencies:
  - Database Query Time
  - External API Calls (payment gateway)
  - Cache Hit Rate
  
Resources:
  - CPU per Container
  - Memory per Container
  - Network I/O
  
Business Metrics:
  - Transaction Amount (sum)
  - Average Transaction Size
  - Payment Methods Distribution
  
Traces & Logs:
  - Slow Traces (> 1s)
  - Error Logs Stream
  - Deployment Events
```

---

## 📝 Tóm Tắt

### **Key Decisions:**

```
1. Dashboard Type
   Timeboard: Troubleshooting, correlation
   Screenboard: Overview, mixed timeframes
   
2. Widget Selection
   Match widget to data type
   Don't force wrong widget
   
3. Query Design
   Aggregate appropriately
   Use template variables
   Add context (thresholds, notes)
   
4. Layout
   Prioritize: Top = most important
   Group related metrics
   Consistent sizing
   
5. Audience
   Different dashboards for different audiences
   Executive ≠ Engineer dashboards
```

### **Best Practices:**

```
✅ Purpose-built dashboards
✅ Clear naming conventions
✅ Add context (notes, thresholds)
✅ Use template variables
✅ Conditional formatting
✅ Limit widgets (10-30 per dashboard)
✅ Document in notes widgets
```

### **Banking Specifics:**

```
✅ SLO widgets prominent
✅ Business metrics included
✅ Compliance/audit trail
✅ Multi-region views
✅ Separate dashboards for different stakeholders
```

---

## ➡️ Bước Tiếp Theo

**Related Topics:**
- [06 - Metrics Design](06-METRICS.md) - Metric types, naming
- [12 - Monitors & Alerts](12-MONITORS-ALERTS.md) - Alert from dashboard metrics
- [13 - SLO](13-SLO.md) - SLO widgets and tracking

---

**📌 Ghi Chú Của Bạn**
```
(Dashboard designs, widget decisions, lessons learned)








```
