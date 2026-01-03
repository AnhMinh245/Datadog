# 05 - DASHBOARD VÀ VISUALIZATION

## 🎯 Mục Tiêu Bài Học
Sau bài học này, bạn sẽ:
- Hiểu các loại dashboard và widgets
- Tạo dashboard tùy chỉnh
- Viết queries hiệu quả
- Chia sẻ và export dashboards
- Apply best practices

---

## 📊 Dashboard Types

### 1. Timeboard
> Dashboard với timeline đồng bộ cho tất cả widgets

**Đặc điểm:**
```
✅ Shared time frame across all widgets
✅ Zoom và pan timeline
✅ Tốt cho troubleshooting
✅ Compare multiple metrics trong cùng timeframe
✅ Template variables
```

**Use cases:**
```
- Incident investigation
- Performance analysis
- System correlation
- Root cause analysis
```

### 2. Screenboard
> Dashboard với layout tự do và flexible timing

**Đặc điểm:**
```
✅ Mỗi widget có independent timeframe
✅ Drag-and-drop layout
✅ Widgets có thể resize
✅ Tốt cho high-level overview
✅ Embedded content (iframe, images)
```

**Use cases:**
```
- Executive dashboards
- Status boards
- Team overviews
- Public displays
```

### So Sánh

| Feature | Timeboard | Screenboard |
|---------|-----------|-------------|
| **Time synchronization** | Global | Per widget |
| **Layout** | Grid-based | Free-form |
| **Troubleshooting** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Presentation** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Template variables** | Yes | Yes |
| **Auto-refresh** | Yes | Yes |

---

## 🧩 Widget Types

### 1. **Timeseries**
> Line/Area/Bar chart theo thời gian

```
Use cases:
- CPU usage over time
- Request rate trends
- Memory consumption
- Response time evolution
```

**Example Query:**
```
avg:system.cpu.user{env:production} by {host}
```

**Visualization Options:**
```
- Line (default)
- Area
- Bars
- Display: lines, bands, both
- Interpolation: linear, step, cardinal
```

### 2. **Query Value**
> Single number - Latest value or calculation

```
Use cases:
- Current CPU %
- Total requests today
- Active users count
- Error rate
```

**Example:**
```
Query: avg:system.cpu.user{host:web-01}
Result: 67.5%

With conditional formatting:
< 50%  → Green
50-80% → Yellow
> 80%  → Red
```

### 3. **Top List**
> Ranking list - Top N items

```
Use cases:
- Top 10 hosts by CPU
- Most requested endpoints
- Largest tables
- Busiest services
```

**Example:**
```
Query: avg:system.cpu.user{*} by {host}
Display: Top 10 hosts
Order: Descending
```

### 4. **Heatmap**
> Color-coded grid showing distribution

```
Use cases:
- Response time distribution
- Error patterns across hosts
- Request volume by hour/day
- Latency percentiles
```

**Example:**
```
Query: avg:trace.http.request.duration{*} by {service}
Shows: How response times distributed across services
```

### 5. **Distribution**
> Histogram or percentile distribution

```
Use cases:
- Latency percentiles (p50, p95, p99)
- Request size distribution
- Query duration spread
```

**Example:**
```
Query: p95:trace.http.request.duration{service:api}
Shows: 95th percentile response time
```

### 6. **Log Stream**
> Real-time log flow

```
Use cases:
- Live error monitoring
- Deployment watching
- Real-time debugging
```

**Configuration:**
```
Query: source:python status:error
Columns: timestamp, service, message
Limit: Last 50 logs
```

### 7. **Host Map**
> Visual grid of hosts colored by metric

```
Use cases:
- Infrastructure overview
- Quick health check
- Anomaly spotting
```

**Example:**
```
Metric: avg:system.cpu.user{*} by {host}
Color: Green (low) → Red (high)
Group by: availability-zone
```

### 8. **Service Map**
> Dependency graph for services

```
Use cases:
- Microservices topology
- Service dependencies
- Performance bottlenecks
```

**Auto-generated from:**
```
APM traces
Service-to-service calls
Shows: Request volume, latency, error rate
```

### 9. **Table**
> Tabular data with multiple columns

```
Use cases:
- Multi-metric comparison
- Detailed host information
- Resource inventory
```

**Example:**
```
Rows: Hosts
Columns:
  - CPU avg
  - Memory avg
  - Disk usage
  - Network in/out
```

### 10. **SLO Widget**
> Service Level Objective status

```
Shows:
- SLO target: 99.9%
- Current: 99.95%
- Error budget remaining
- Time period
```

---

## 🔍 Query Language

### Query Syntax

**Basic Format:**
```
<aggregation>:<metric>{<filters>} [by {<tags>}]
```

**Components:**

#### 1. Aggregation
```
avg   - Average value
sum   - Sum all values
min   - Minimum value
max   - Maximum value
count - Count of points
```

**Examples:**
```
avg:system.cpu.user{*}
sum:http.requests{*}
max:database.connections{*}
```

#### 2. Metric Name
```
Format: <namespace>.<name>
Examples:
  system.cpu.user
  http.requests
  database.query.time
  custom.business.metric
```

#### 3. Filters (Tags)
```
{tag_key:tag_value}
{tag1:value1,tag2:value2}  # AND logic
{tag:value1 OR tag:value2}  # OR logic
```

**Examples:**
```
{env:production}
{env:production,service:api}
{host:web-01 OR host:web-02}
{env:production,service:api OR service:worker}
```

#### 4. Group By
```
by {tag_key}
by {tag1,tag2}
```

**Examples:**
```
avg:system.cpu.user{*} by {host}
sum:http.requests{*} by {service,status_code}
```

### Advanced Queries

#### Arithmetic
```
# CPU average across all cores
avg:system.cpu.user{*} + avg:system.cpu.system{*}

# Free memory percentage
(avg:system.mem.total{*} - avg:system.mem.used{*}) / avg:system.mem.total{*} * 100

# Request rate per second
sum:http.requests{*}.as_count() / 60
```

#### Functions

**Rate:**
```
# Convert count to rate
sum:http.requests{*}.as_rate()

# Result: requests per second
```

**Rollup:**
```
# Custom time aggregation
avg:system.cpu.user{*}.rollup(avg, 60)

# Average CPU over 60 second windows
```

**Timeshift:**
```
# Compare with yesterday
avg:system.cpu.user{*}
timeshift(avg:system.cpu.user{*}, 86400)

# 86400 seconds = 1 day
```

**Anomalies:**
```
# Detect anomalies
anomalies(avg:system.cpu.user{*}, 'basic', 2)

# Algorithm: basic, agile, robust
# Bounds: 2 = 2 standard deviations
```

**Forecast:**
```
# Predict future values
forecast(avg:system.disk.used{*}, 'linear', 1d)

# Algorithm: linear, seasonal
# Duration: 1d = 1 day ahead
```

**Smoothing:**
```
# Exponentially weighted moving average
ewma_5(avg:system.cpu.user{*})

# Options: ewma_3, ewma_5, ewma_10, ewma_20
```

---

## 🎨 Creating Dashboards

### Step-by-Step: Create Dashboard

**1. Navigate:**
```
Dashboards → New Dashboard
→ Choose: New Timeboard or New Screenboard
```

**2. Add Widget:**
```
Click: + Add Widget
Select widget type
Configure query
Customize visualization
Set title
Save
```

**3. Example: System Overview Dashboard**

```yaml
Dashboard: "Production System Overview"
Type: Timeboard
Timeframe: Past 1 Hour

Widgets:
  - Row 1:
    - CPU Usage (Timeseries)
      Query: avg:system.cpu.user{env:production} by {host}
      Viz: Lines
      
    - Memory Usage (Timeseries)
      Query: avg:system.mem.used{env:production} by {host}
      Viz: Area
      
  - Row 2:
    - Request Rate (Timeseries)
      Query: sum:http.requests{env:production}.as_rate()
      Viz: Bars
      
    - Error Rate (Timeseries)
      Query: sum:http.errors{env:production}.as_rate()
      Viz: Lines (Red)
      
  - Row 3:
    - Top Hosts by CPU (Top List)
      Query: avg:system.cpu.user{env:production} by {host}
      Limit: 10
      
    - Response Time p95 (Query Value)
      Query: p95:trace.http.request.duration{env:production}
      Unit: ms
```

### Template Variables

**Purpose:**
```
Dynamic filtering without editing queries
Reusable dashboards
Easy switching between environments
```

**Example Setup:**

```
Variable Name: $env
Tag: env
Default: production
Values: production, staging, development

Variable Name: $service
Tag: service
Default: *
Values: api, web, worker, database

Variable Name: $host
Tag: host
Default: *
Values: (Auto-populated)
```

**Usage in Queries:**
```
# Before
avg:system.cpu.user{env:production,service:api}

# After (with template variables)
avg:system.cpu.user{env:$env,service:$service}

# User can now select values from dropdowns
```

**Advanced Variable:**
```
Variable Name: $aggregation
Type: Text input
Default: avg
Usage: $aggregation:system.cpu.user{env:$env}

# Allows user to choose aggregation method
```

---

## 🎯 Dashboard Best Practices

### 1. Organization

```
✅ Group related metrics together
✅ Use rows/sections with headers
✅ Order by importance (top = most critical)
✅ Consistent time frames
✅ Limit widgets (10-15 per dashboard)

❌ Don't overcrowd
❌ Don't mix unrelated metrics
❌ Don't use inconsistent time ranges
```

### 2. Naming Conventions

```
✅ Clear, descriptive names
   "Production API - Performance Overview"
   "Database - MySQL Cluster Health"

❌ Vague names
   "Dashboard 1"
   "My Dashboard"
```

### 3. Color Coding

```
✅ Green: Good/Normal
✅ Yellow: Warning
✅ Red: Critical/Error
✅ Blue: Info/Neutral

Consistent across all dashboards
```

### 4. Context

```
✅ Add notes/text widgets explaining metrics
✅ Document thresholds
✅ Link to runbooks
✅ Add business context

Example Note Widget:
"Target: < 200ms response time
Alert threshold: > 500ms
Runbook: https://wiki.company.com/api-slow"
```

### 5. Responsive Design

```
✅ Test on different screen sizes
✅ Use appropriate widget sizes
✅ Important metrics → Larger widgets
✅ Details → Smaller widgets
```

---

## 📤 Sharing & Export

### Share Dashboard

**1. Public URL:**
```
Dashboard → Share → Generate Public URL
→ Anyone with link can view (read-only)
→ Set expiration time
→ Revoke anytime
```

**2. Email/Slack:**
```
Dashboard → Share → Send Snapshot
→ Choose recipients
→ Add message
→ Schedule recurring (optional)
```

**3. Embed:**
```
Dashboard → Share → Generate Embed Code
→ Copy iframe code
→ Paste in wiki/docs
→ Requires authentication
```

### Export

**JSON:**
```
Dashboard → Settings → Export Dashboard JSON
→ Can import to other accounts
→ Version control in git
→ Infrastructure as Code
```

**PDF:**
```
Dashboard → Generate PDF
→ Select timeframe
→ Download or email
→ Good for reports
```

**API:**
```python
from datadog import initialize, api

initialize()

# Get dashboard
dashboard = api.Dashboard.get('dashboard-id')

# Create dashboard from JSON
api.Dashboard.create(
    title='My Dashboard',
    widgets=[...],
    layout_type='ordered'
)
```

---

## 📺 Monitoring Dashboard on TV

### Setup TV Display

**1. Full Screen Mode:**
```
Dashboard → Settings → TV Mode
→ Hides navigation
→ Auto-refresh
→ Optimized for large screens
```

**2. Kiosk Mode:**
```
URL parameter: ?kiosk=true
Example: https://app.datadoghq.com/dashboard/abc?kiosk=true

Features:
- No navigation bar
- No edit controls
- Clean view
```

**3. Rotation:**
```
Create multiple dashboards
Use browser extension for tab rotation
Example: Tab Rotator, Revolver

Setup:
- Dashboard 1: System Overview (30 sec)
- Dashboard 2: Application Metrics (30 sec)
- Dashboard 3: Error Tracking (20 sec)
- Repeat
```

---

## 🔧 Advanced Features

### 1. Dashboard Lists
```
Organize dashboards into lists
Share entire list with team
Example lists:
- "Infrastructure Team Dashboards"
- "On-Call Dashboards"
- "Executive Dashboards"
```

### 2. Conditional Formatting
```
Query Value Widget:
Value < 50%  → Green background
Value 50-80% → Yellow background
Value > 80%  → Red background + Alert icon
```

### 3. Event Overlays
```
Show deployments, alerts, incidents on timeline

Example:
Timeseries of CPU usage
+ Overlay: Deployment events
→ See if deployment caused CPU spike
```

### 4. Custom Links
```
Widget → Add Custom Links

Example:
Click on error count →
Jumps to: Logs Explorer filtered by error status
```

### 5. Mobile App
```
iOS/Android: Datadog Mobile App
Features:
- View dashboards
- Receive alerts
- Acknowledge incidents
- Quick actions
```

---

## 📝 Example Dashboards

### Dashboard 1: Web Application Health

```yaml
Name: "Production Web App - Health Dashboard"
Type: Timeboard
Auto-refresh: 1 minute

Layout:
  Row 1 - Overview:
    - Total Requests (Query Value)
      Query: sum:http.requests{env:production}.as_count()
      Timeframe: Last 1 hour
      
    - Error Rate % (Query Value)
      Query: (sum:http.errors{*} / sum:http.requests{*}) * 100
      Conditional: > 1% = Red
      
    - Avg Response Time (Query Value)
      Query: avg:trace.http.request.duration{*}
      Unit: ms
      
    - Active Users (Query Value)
      Query: count_nonzero:users.active{*}
      
  Row 2 - Traffic:
    - Requests per Minute (Timeseries)
      Query: sum:http.requests{env:production}.as_rate()
      Group by: endpoint
      
    - Response Codes (Timeseries - Stacked)
      Queries:
        - sum:http.requests{status:2xx}
        - sum:http.requests{status:4xx}
        - sum:http.requests{status:5xx}
        
  Row 3 - Performance:
    - Response Time p95 (Timeseries)
      Query: p95:trace.http.request.duration{*}
      
    - Slow Endpoints (Top List)
      Query: avg:trace.http.request.duration{*} by {endpoint}
      Order: Desc, Limit: 10
      
  Row 4 - Infrastructure:
    - CPU Usage (Timeseries)
      Query: avg:system.cpu.user{role:web} by {host}
      
    - Memory Usage (Timeseries)
      Query: avg:system.mem.used_pct{role:web} by {host}
```

### Dashboard 2: Database Monitoring

```yaml
Name: "PostgreSQL Performance"
Type: Timeboard

Widgets:
  - Connection Pool (Timeseries)
    Query: avg:postgresql.connections{*}
    
  - Slow Queries (Log Stream)
    Query: source:postgresql @duration:>1000
    
  - Query Rate (Timeseries)
    Query: sum:postgresql.queries{*}.as_rate()
    
  - Deadlocks (Timeseries)
    Query: sum:postgresql.deadlocks{*}
    Alert: > 0
    
  - Table Sizes (Top List)
    Query: max:postgresql.table.size{*} by {table}
    
  - Cache Hit Ratio (Query Value)
    Query: avg:postgresql.cache_hit_ratio{*}
    Target: > 99%
```

---

## 🤔 Dashboard Design Decisions

### System monitoring dashboard:

```
Tạo dashboard cho system monitoring:

Required Widgets:
1. CPU Usage (timeseries) - last 4 hours
2. Memory Usage (timeseries) - last 4 hours
3. Disk Usage (query value) - current
4. Top 5 processes by CPU (top list)
5. Network Traffic In/Out (timeseries)

Template Variables:
- $host
- $env

Bonus:
- Add conditional formatting
- Add note widget with thresholds
```

### Application monitoring dashboard:

```
Tạo dashboard cho web application:

Required:
1. Request rate (last 1 hour)
2. Error rate %
3. Response time (p50, p95, p99)
4. Top endpoints by traffic
5. Error log stream

Advanced:
- Compare với yesterday (timeshift)
- Overlay deployment events
- Add custom links to logs
```

### Template variables strategy:

```
Tạo dashboard với template variables:

Variables:
1. $env (production, staging)
2. $service (api, web, worker)
3. $region (us-east-1, us-west-2, eu-west-1)

All widgets phải dùng variables
Test bằng cách switch giữa các values
```

### Sharing & export considerations:

```
Tasks:
1. Export dashboard to JSON
2. Commit JSON to git
3. Generate public URL (24h expiration)
4. Create scheduled snapshot (send daily via email)
5. Embed dashboard in Confluence/Wiki
```

---

## 📚 Resources

### Official Docs
```
- Dashboard Overview: https://docs.datadoghq.com/dashboards/
- Widget Reference: https://docs.datadoghq.com/dashboards/widgets/
- Query Functions: https://docs.datadoghq.com/dashboards/functions/
```

### Video Tutorials
```
- Datadog Dashboard Basics (YouTube)
- Advanced Querying Techniques
- Template Variables Deep Dive
```

---

## 📝 Tóm Tắt

### Key Concepts

```
1. DASHBOARD TYPES
   - Timeboard: Shared timeline, troubleshooting
   - Screenboard: Flexible layout, presentations

2. WIDGET TYPES
   - Timeseries, Query Value, Top List
   - Heatmap, Distribution, Log Stream
   - Service Map, Table, SLO

3. QUERY SYNTAX
   aggregation:metric{filters} by {tags}

4. TEMPLATE VARIABLES
   Dynamic filtering, reusable dashboards

5. BEST PRACTICES
   - Clear organization
   - Consistent naming
   - Appropriate colors
   - Context & documentation
```

---

## ➡️ Bước Tiếp Theo

Bạn đã biết cách tạo và sử dụng dashboards!

**Bài tiếp theo**: [06 - Metrics và Monitoring](06-METRICS.md)

Trong bài tiếp theo, chúng ta sẽ học sâu về metrics, custom metrics, và monitoring strategies.

---

**📌 Ghi Chú Của Bạn**
```
(Dashboard IDs, query examples, tips & tricks)








```

