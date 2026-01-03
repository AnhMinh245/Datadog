# 06 - METRICS VÀ MONITORING

## 🎯 Mục Tiêu Bài Học
Sau bài học này, bạn sẽ:
- Hiểu sâu về metrics và types
- Gửi custom metrics từ application
- Tối ưu metrics collection
- Monitor hiệu quả với metrics
- Best practices cho metrics

---

## 📊 Metrics Deep Dive

### Metric Types Chi Tiết

#### 1. **GAUGE**
> Giá trị tại một thời điểm - Có thể tăng/giảm

**Đặc điểm:**
```
- Snapshot value
- Có thể thay đổi tự do
- Datadog lưu giá trị cuối cùng trong flush interval
```

**Examples:**
```
system.cpu.user = 75.3%
system.mem.used = 8.5 GB
database.connections = 45
queue.size = 1250
temperature = 23.5°C
```

**Khi nào dùng:**
```
✅ Current state của system
✅ Resource levels
✅ Inventory counts
✅ Percentages
✅ Temperatures, speeds

❌ Events hoặc occurrences
❌ Cumulative counts
```

**Code Example:**
```python
from datadog import statsd

# Simple gauge
statsd.gauge('users.active', 1250)

# Gauge with tags
statsd.gauge(
    'database.connections',
    45,
    tags=['env:production', 'db:main']
)

# Increment/Decrement gauge
statsd.gauge('queue.size', 100)  # Set to 100
# Later...
current = get_queue_size()
statsd.gauge('queue.size', current)  # Update
```

#### 2. **COUNT**
> Số lần event xảy ra trong interval

**Đặc điểm:**
```
- Tích lũy trong flush interval (15s default)
- Reset về 0 sau mỗi flush
- Datadog sum các counts trong interval
```

**Examples:**
```
http.requests = 500 (trong 15s)
errors.occurred = 3
orders.completed = 10
emails.sent = 25
```

**Khi nào dùng:**
```
✅ Events happening
✅ Counts of occurrences
✅ Things that happen over time

❌ Current state
❌ Cumulative totals across time
```

**Code Example:**
```python
from datadog import statsd

# Increment by 1
statsd.increment('page.views')

# Increment by custom value
statsd.increment('api.requests', 5)

# With tags
statsd.increment(
    'order.completed',
    tags=['country:us', 'payment:stripe']
)

# Decrement
statsd.decrement('items.in_stock')
```

#### 3. **RATE**
> Count per second

**Đặc điểm:**
```
- Automatic conversion from count
- Unit: per second
- Normalized across time
```

**Examples:**
```
http.requests.rate = 33.3 requests/second
errors.rate = 0.2 errors/second
```

**Conversion:**
```
500 requests in 15 seconds
= 500 / 15
= 33.3 requests/second
```

**Code Example:**
```python
# Send as count, visualize as rate
statsd.increment('api.requests')

# In Datadog query:
sum:api.requests{*}.as_rate()

# Result: requests per second
```

#### 4. **HISTOGRAM/DISTRIBUTION**
> Statistical distribution of values

**Đặc điểm:**
```
- Calculates: avg, count, median, p95, p99, max
- Client-side aggregation (Histogram)
- Server-side aggregation (Distribution)
```

**Examples:**
```
request.duration = [100ms, 150ms, 200ms, 120ms, ...]
→ avg: 142ms
→ p50: 135ms
→ p95: 198ms
→ p99: 200ms
→ max: 200ms
```

**Histogram vs Distribution:**

| Feature | Histogram | Distribution |
|---------|-----------|-------------|
| **Aggregation** | Client-side (Agent) | Server-side (Datadog) |
| **Percentiles** | p50, p75, p95, p99 | Any percentile |
| **Accuracy** | Approximate | Exact |
| **Cost** | Lower | Higher |
| **Flexibility** | Fixed percentiles | Query-time percentiles |

**Code Example:**
```python
from datadog import statsd
import time

# Histogram
start = time.time()
# ... do work ...
duration = time.time() - start
statsd.histogram('request.duration', duration * 1000)  # ms

# Distribution
statsd.distribution('request.size', len(response_body))

# With tags
statsd.histogram(
    'database.query.time',
    query_duration,
    tags=['query_type:select', 'table:users']
)
```

---

## 📨 Sending Custom Metrics

### Method 1: DogStatsD (Recommended)

**Setup:**

```python
# Install
pip install datadog

# Initialize
from datadog import initialize, statsd

options = {
    'statsd_host': '127.0.0.1',
    'statsd_port': 8125,
}
initialize(**options)
```

**Usage Examples:**

```python
from datadog import statsd

# Gauge - Current value
statsd.gauge('users.online', 1250)

# Counter - Increment events
statsd.increment('orders.placed')
statsd.increment('revenue', 99.99)

# Histogram - Duration/Size distributions
statsd.histogram('request.duration', 245)  # ms
statsd.histogram('image.size', 1024000)    # bytes

# Distribution
statsd.distribution('response.size', response_len)

# Timing (special histogram for durations)
statsd.timing('database.query', 50)  # ms

# Set - Count unique elements
statsd.set('unique.visitors', user_id)

# Tags
statsd.increment(
    'api.request',
    tags=['endpoint:/users', 'method:GET', 'status:200']
)

# Sample rate (send only 10% of metrics)
statsd.increment('high.volume.metric', sample_rate=0.1)
```

**Context Manager for Timing:**

```python
from datadog import statsd

# Automatic timing
with statsd.timed('my_function.duration'):
    # Code to time
    result = expensive_operation()

# Equivalent to:
start = time.time()
result = expensive_operation()
duration = time.time() - start
statsd.timing('my_function.duration', duration * 1000)
```

**Decorator:**

```python
from datadog import statsd

@statsd.timed('user.login.duration')
def login_user(username, password):
    # Login logic
    return user

# Auto sends metric: user.login.duration
```

### Method 2: HTTP API

```python
import requests
import time

def send_metric(api_key, metric_name, value, tags=None):
    url = 'https://api.datadoghq.com/api/v1/series'
    
    headers = {
        'Content-Type': 'application/json',
        'DD-API-KEY': api_key
    }
    
    payload = {
        'series': [{
            'metric': metric_name,
            'points': [[int(time.time()), value]],
            'type': 'gauge',
            'tags': tags or []
        }]
    }
    
    response = requests.post(url, json=payload, headers=headers)
    return response.json()

# Usage
send_metric(
    api_key='YOUR_API_KEY',
    metric_name='custom.business.metric',
    value=100,
    tags=['env:production', 'team:sales']
)
```

### Method 3: Agent Check (Advanced)

```python
# /etc/datadog-agent/checks.d/my_check.py

from datadog_checks.base import AgentCheck

class MyCheck(AgentCheck):
    def check(self, instance):
        # Custom logic
        value = self.get_my_metric()
        
        # Send gauge
        self.gauge(
            'my.custom.metric',
            value,
            tags=['env:production']
        )
        
        # Send counter
        self.count('my.events', 1)
        
        # Send service check
        self.service_check(
            'my.service.status',
            AgentCheck.OK,
            message='Service is healthy'
        )
    
    def get_my_metric(self):
        # Custom collection logic
        return 42
```

```yaml
# /etc/datadog-agent/conf.d/my_check.d/conf.yaml
init_config:

instances:
  - min_collection_interval: 60
    tags:
      - env:production
```

---

## 📐 Metric Naming Conventions

### Best Practices

**Format:**
```
<namespace>.<entity>.<action>.<attribute>

Examples:
✅ web.requests.count
✅ database.query.duration
✅ cache.hit.rate
✅ payment.transaction.amount

❌ requests
❌ db_query_time
❌ CacheHitRate
```

**Rules:**
```
✅ Lowercase
✅ Dot-separated
✅ Descriptive
✅ Consistent
✅ Max 200 characters

❌ Uppercase
❌ Special characters (@, #, $)
❌ Spaces
❌ High cardinality in name
```

**Namespace Examples:**

```
Application Metrics:
  app.requests.*
  app.errors.*
  app.users.*

Infrastructure:
  system.cpu.*
  system.memory.*
  system.disk.*

Business Metrics:
  business.revenue.*
  business.conversions.*
  business.signups.*

Custom Services:
  payment.transactions.*
  email.sent.*
  queue.messages.*
```

---

## 🏷️ Tags Strategy

### Tag Dimensions

```
Infrastructure Tags:
  env:production
  region:us-east-1
  availability_zone:us-east-1a
  host:web-server-01
  instance_type:m5.large

Application Tags:
  service:api
  version:v2.1.0
  team:backend
  component:payment

Request Tags:
  endpoint:/api/users
  method:GET
  status_code:200
  user_tier:premium

Business Tags:
  country:us
  plan:enterprise
  feature:new_checkout
```

### Cardinality Management

**Low Cardinality (Good):**
```
✅ env: prod, staging, dev (3 values)
✅ service: api, web, worker (10 values)
✅ region: us-east-1, us-west-2 (5 values)
✅ status: 200, 400, 500 (10 values)

Total combinations: Manageable
```

**High Cardinality (Bad):**
```
❌ user_id: 1, 2, 3, ..., 1000000
❌ request_id: uuid-1, uuid-2, ...
❌ timestamp: 1234567890, 1234567891, ...
❌ ip_address: 192.168.1.1, 192.168.1.2, ...

Total combinations: Millions
→ Expensive
→ Slow queries
→ May hit limits
```

**Solution for High Cardinality:**
```
Don't use as tags → Use in logs instead

Instead of:
  statsd.increment('request', tags=[f'user_id:{user_id}'])

Do:
  statsd.increment('request', tags=['user_tier:premium'])
  logger.info(f'Request from user {user_id}')
```

---

## 📊 Monitoring Patterns

### Pattern 1: RED Method
> Rate, Errors, Duration - For services

```python
# Rate - Requests per second
statsd.increment('api.requests', tags=['endpoint:/users'])

# Errors - Error rate
if error:
    statsd.increment('api.errors', tags=['endpoint:/users', 'error_type:timeout'])

# Duration - Response time
statsd.histogram('api.duration', duration_ms, tags=['endpoint:/users'])
```

**Dashboard Queries:**
```
Rate: sum:api.requests{*}.as_rate() by {endpoint}
Errors: sum:api.errors{*}.as_rate() by {endpoint}
Duration: p95:api.duration{*} by {endpoint}
```

### Pattern 2: USE Method
> Utilization, Saturation, Errors - For resources

```python
# Utilization - % of resource used
cpu_percent = psutil.cpu_percent()
statsd.gauge('system.cpu.utilization', cpu_percent)

# Saturation - Queue depth/wait time
queue_size = get_queue_size()
statsd.gauge('system.queue.saturation', queue_size)

# Errors - Resource errors
if disk_error:
    statsd.increment('system.disk.errors')
```

### Pattern 3: Four Golden Signals
> Latency, Traffic, Errors, Saturation

```python
# Latency
statsd.histogram('service.latency', duration)

# Traffic
statsd.increment('service.requests')

# Errors
if error:
    statsd.increment('service.errors')

# Saturation
cpu = psutil.cpu_percent()
statsd.gauge('service.cpu.saturation', cpu)
```

---

## 📈 Business Metrics

### E-commerce Example

```python
from datadog import statsd

class EcommerceMetrics:
    
    @staticmethod
    def track_order_placed(order_value, user_country, payment_method):
        """Track new order"""
        statsd.increment(
            'business.orders.placed',
            tags=[
                f'country:{user_country}',
                f'payment:{payment_method}'
            ]
        )
        
        statsd.histogram(
            'business.order.value',
            order_value,
            tags=[f'country:{user_country}']
        )
    
    @staticmethod
    def track_user_signup(plan):
        """Track new user signup"""
        statsd.increment(
            'business.users.signup',
            tags=[f'plan:{plan}']
        )
    
    @staticmethod
    def track_revenue(amount, currency='USD'):
        """Track revenue"""
        statsd.histogram(
            'business.revenue',
            amount,
            tags=[f'currency:{currency}']
        )
    
    @staticmethod
    def track_cart_abandonment():
        """Track cart abandonment"""
        statsd.increment('business.cart.abandoned')
    
    @staticmethod
    def track_product_view(product_id, category):
        """Track product views"""
        statsd.increment(
            'business.product.views',
            tags=[f'category:{category}']
        )

# Usage
EcommerceMetrics.track_order_placed(
    order_value=99.99,
    user_country='US',
    payment_method='stripe'
)

EcommerceMetrics.track_revenue(99.99, currency='USD')
```

### SaaS Metrics

```python
class SaaSMetrics:
    
    @staticmethod
    def track_active_users(count):
        """Daily/Monthly Active Users"""
        statsd.gauge('business.users.active.daily', count)
    
    @staticmethod
    def track_feature_usage(feature_name, user_plan):
        """Feature adoption"""
        statsd.increment(
            'business.feature.usage',
            tags=[
                f'feature:{feature_name}',
                f'plan:{user_plan}'
            ]
        )
    
    @staticmethod
    def track_churn(plan):
        """User churn"""
        statsd.increment(
            'business.churn',
            tags=[f'plan:{plan}']
        )
    
    @staticmethod
    def track_mrr(amount):
        """Monthly Recurring Revenue"""
        statsd.gauge('business.mrr', amount)
```

---

## 🎯 Monitoring Strategy

### Metrics Hierarchy

```
Level 1: Infrastructure (Always monitor)
  - CPU, Memory, Disk, Network
  - Host health
  - Container health

Level 2: Application (Service health)
  - Request rate
  - Error rate
  - Response time
  - Availability

Level 3: Business (Business health)
  - Revenue
  - User signups
  - Conversions
  - Feature usage

Level 4: User Experience (End-user view)
  - Page load time
  - Transaction success rate
  - User session duration
```

### What to Monitor

```
Golden Rules:
1. Monitor symptoms, not causes
   ✅ Response time is slow (symptom)
   ❌ Database has 100 connections (cause)

2. Monitor user-facing metrics
   ✅ API response time
   ❌ Background job duration (unless affects users)

3. Monitor what you can act on
   ✅ Disk 90% full → Add disk
   ❌ Network packet count → Can't control

4. Monitor business impact
   ✅ Failed payments → Lost revenue
   ✅ Slow checkout → Lower conversion
```

---

## 💰 Cost Optimization

### Metrics Pricing
```
Datadog pricing:
- $0.05 per custom metric (host-based)
- Custom metric = unique combination of:
  metric name + tag set

Example:
api.requests{env:prod,service:web,region:us}
→ 1 custom metric

If you add status tag with 5 values (200,201,400,404,500):
api.requests{env:prod,service:web,region:us,status:200}
api.requests{env:prod,service:web,region:us,status:201}
...
→ 5 custom metrics
→ $0.25/month
```

### Optimization Tips

**1. Reduce Tag Cardinality:**
```
❌ Before: (100 hosts × 20 services × 10 status codes)
   = 20,000 custom metrics

✅ After: Group status codes
   status:2xx instead of status:200,201,202...
   = 100 hosts × 20 services × 3 groups
   = 6,000 custom metrics
```

**2. Sample High-Volume Metrics:**
```python
# Send only 10% of metrics
statsd.increment(
    'high.volume.metric',
    sample_rate=0.1
)
# Datadog multiplies by 10 to approximate
```

**3. Use Distributions Wisely:**
```
Distribution = More expensive than Histogram
Use only when you need query-time percentiles
```

**4. Aggregate Before Sending:**
```python
# ❌ Bad: Send every request
for request in requests:
    statsd.increment('requests')

# ✅ Good: Aggregate then send
request_count = len(requests)
statsd.increment('requests', request_count)
```

**5. Use Metric Type Efficiently:**
```
✅ Gauge for current state (cheap)
✅ Count for events (cheap)
❌ Distribution for everything (expensive)
```

---

## 📝 Tóm Tắt

### Key Takeaways

```
1. METRIC TYPES
   - Gauge: Current value
   - Count: Events over time
   - Rate: Events per second
   - Histogram/Distribution: Statistical distribution

2. SENDING METRICS
   - DogStatsD (recommended for apps)
   - HTTP API (programmatic)
   - Agent Check (custom integrations)

3. NAMING CONVENTIONS
   - namespace.entity.action.attribute
   - Lowercase, dot-separated
   - Descriptive, consistent

4. TAGS
   - Low cardinality
   - Meaningful dimensions
   - Consistent across services

5. MONITORING PATTERNS
   - RED: Rate, Errors, Duration
   - USE: Utilization, Saturation, Errors
   - Four Golden Signals

6. COST OPTIMIZATION
   - Reduce tag cardinality
   - Sample high-volume metrics
   - Choose right metric types
```

---

## 🤔 Metrics Design Decisions

### Custom metrics implementation:

```python
# Tạo script gửi custom metrics

Requirements:
1. Install datadog package
2. Send gauge: current_time (timestamp)
3. Send counter: script_runs
4. Send histogram: random_duration (0-1000ms)
5. Add tags: env:lab, student:your-name

Run script 10 times
Verify metrics trong UI
```

### Business metrics strategy:

```python
# Simulate e-commerce metrics

Track:
1. Orders placed (counter)
2. Order value (histogram)
3. Active users (gauge)
4. Cart abandonment rate (count)

Tags:
- country (US, UK, DE)
- payment_method (stripe, paypal)
- user_tier (free, premium)

Send random data for 5 minutes
Create dashboard to visualize
```

### Monitoring pattern application:

```
Implement RED method for fake API:

Metrics:
1. api.requests (rate)
2. api.errors (rate)
3. api.duration (p95)

Tags:
- endpoint: /users, /products, /orders
- method: GET, POST
- status: 200, 400, 500

Create dashboard showing:
- Request rate by endpoint
- Error rate %
- p95 latency by endpoint
```

### Cost analysis:

```
Calculate custom metric count:

System:
- 10 hosts
- 5 services per host
- 3 environments (prod, staging, dev)
- 3 regions

Metric: api.requests
Tags: host, service, env, region

Questions:
1. How many unique tag combinations?
2. Monthly cost at $0.05/metric?
3. How to reduce by 50%?
```

---

## ➡️ Bước Tiếp Theo

Bạn đã master metrics!

**Bài tiếp theo**: [07 - APM (Application Performance Monitoring)](07-APM.md)

Trong bài tiếp theo, chúng ta sẽ học về APM để trace requests qua distributed systems.

---

**📌 Ghi Chú Của Bạn**
```
(Code snippets, metric names, calculations)








```

