# 07 - APM (APPLICATION PERFORMANCE MONITORING)

## 🎯 Mục Tiêu Bài Học
Sau bài học này, bạn sẽ:
- Hiểu APM và distributed tracing
- Instrument application với Datadog tracer
- Analyze performance bottlenecks
- Optimize application performance
- Troubleshoot issues với traces

---

## 🔍 APM Là Gì?

### Definition
> **APM (Application Performance Monitoring)** theo dõi performance của application bằng cách trace requests qua distributed system

**Giải thích đơn giản:**
```
Giống như GPS tracking cho một package:
- Package (Request) đi qua nhiều điểm (Services)
- Biết được đi qua đâu, mất bao lâu ở mỗi điểm
- Phát hiện điểm nào bị delay
```

### Core Concepts

**1. Trace**
```
Toàn bộ journey của một request

Example: User clicks "Checkout" button
→ Tạo 1 trace ID
→ Request đi qua Frontend → API → Database → Payment
→ Tất cả thuộc về 1 trace
```

**2. Span**
```
Một operation trong trace

Example trong trace "Checkout":
- Span 1: Frontend processing (50ms)
- Span 2: API validation (20ms)
- Span 3: Database query (100ms)
- Span 4: Payment processing (300ms)
- Span 5: Email sending (150ms)
```

**3. Service**
```
Independent component của application

Examples:
- frontend-web
- api-gateway
- user-service
- payment-service
- email-service
```

**4. Resource**
```
Specific action trong service

Examples trong payment-service:
- POST /payment/process
- GET /payment/status
- PUT /payment/refund
```

---

## 🎨 Trace Visualization

### Flame Graph

```
Request: POST /checkout (Total: 620ms)
│
├─ frontend.render (50ms)
│  └─ Tags: user_id=123, browser=chrome
│
├─ api.gateway (20ms)
│  └─ Tags: endpoint=/checkout, method=POST
│
├─ payment.service (400ms) ⚠️ SLOW
│  ├─ payment.validate_card (30ms)
│  │  └─ Tags: card_type=visa
│  │
│  ├─ payment.call_stripe_api (320ms) ⚠️ BOTTLENECK!
│  │  └─ Tags: stripe_charge_id=ch_123
│  │  └─ Error: None
│  │
│  └─ payment.save_transaction (50ms)
│     └─ Tags: db_query=INSERT INTO transactions
│
└─ email.service (150ms)
   └─ email.send_receipt (150ms)
      └─ Tags: template=receipt, to=user@email.com

TIMELINE:
frontend     [===]
api_gateway    [=]
payment           [====================] ⚠️
  validate          [=]
  stripe_api          [================] ⚠️
  save_tx                  [==]
email                     [======]
├────────────────────────────────────┤
0ms               300ms            620ms
```

### Service Map

```
           ┌──────────┐
           │ Frontend │
           └────┬─────┘
                │ 1000 req/s
                │ 200ms avg
                │ 0.1% errors
                ▼
           ┌──────────┐
           │   API    │
           │ Gateway  │
           └────┬─────┘
                │
         ┌──────┴──────┐
         │             │
         ▼             ▼
    ┌────────┐    ┌─────────┐
    │  User  │    │ Payment │ ⚠️ 450ms avg
    │Service │    │ Service │ ⚠️ 2% errors
    └───┬────┘    └────┬────┘
        │              │
        ▼              ▼
    ┌────────┐    ┌─────────┐
    │Database│    │ Stripe  │
    │        │    │   API   │
    └────────┘    └─────────┘
```

---

## 🛠️ Setup APM

### Prerequisites

```
✅ Datadog Agent installed
✅ APM enabled in Agent config
✅ Tracer library for your language
✅ Application instrumentation
```

### Enable APM in Agent

```yaml
# /etc/datadog-agent/datadog.yaml

apm_config:
  enabled: true
  apm_non_local_traffic: true  # For distributed apps
  
  # Advanced settings
  analyzed_spans:
    # Sample 100% of errors, 50% of other requests
    my-service|*: 0.5
    my-service|error: 1.0
    
  # Filter unwanted traces
  ignore_resources:
    - ^GET /health$
    - ^GET /ping$
```

**Docker:**
```bash
docker run -d \
  -e DD_APM_ENABLED=true \
  -e DD_APM_NON_LOCAL_TRAFFIC=true \
  -p 8126:8126 \
  datadog/agent:latest
```

**Kubernetes:**
```yaml
# datadog-values.yaml
datadog:
  apm:
    portEnabled: true
    socketEnabled: true
```

---

## 🐍 Instrumentation: Python

### Installation

```bash
pip install ddtrace
```

### Automatic Instrumentation

```bash
# Run app with ddtrace
DD_SERVICE=my-app \
DD_ENV=production \
DD_VERSION=1.2.3 \
ddtrace-run python app.py
```

**Frameworks auto-instrumented:**
```
- Flask
- Django
- FastAPI
- Celery
- SQLAlchemy
- Redis
- PostgreSQL
- MongoDB
- requests
- aiohttp
... and more
```

### Manual Instrumentation

```python
from ddtrace import tracer

# Example 1: Trace a function
@tracer.wrap(service='my-service', resource='process_order')
def process_order(order_id):
    # Your code
    return result

# Example 2: Context manager
def checkout(user_id, cart):
    with tracer.trace('checkout.process', service='ecommerce') as span:
        # Add tags
        span.set_tag('user.id', user_id)
        span.set_tag('cart.items', len(cart))
        
        # Process checkout
        result = do_checkout(user_id, cart)
        
        # Add more metadata
        span.set_tag('order.id', result.order_id)
        span.set_tag('order.total', result.total)
        
        return result

# Example 3: Manual span
def payment_process(amount):
    span = tracer.trace('payment.charge')
    span.service = 'payment-service'
    span.resource = 'stripe.charge'
    
    try:
        result = stripe.charge(amount)
        span.set_tag('payment.amount', amount)
        span.set_tag('payment.id', result.id)
        return result
    except Exception as e:
        span.set_error(e)  # Mark span as error
        raise
    finally:
        span.finish()
```

### Flask Example

```python
from flask import Flask
from ddtrace import tracer
from ddtrace.contrib.flask import TraceMiddleware

app = Flask(__name__)

# Auto-instrument Flask
traced_app = TraceMiddleware(
    app,
    tracer,
    service='my-api',
    distributed_tracing=True
)

@app.route('/users/<user_id>')
def get_user(user_id):
    # Automatically traced!
    
    # Add custom tags
    span = tracer.current_span()
    if span:
        span.set_tag('user.id', user_id)
    
    # Your code
    user = User.query.get(user_id)
    return jsonify(user.to_dict())

@app.route('/order', methods=['POST'])
def create_order():
    # Custom span inside route
    with tracer.trace('order.validation') as span:
        data = request.json
        span.set_tag('order.items', len(data['items']))
        validate_order(data)
    
    with tracer.trace('order.save') as span:
        order = Order.create(data)
        span.set_tag('order.id', order.id)
    
    return jsonify(order.to_dict()), 201
```

### Database Instrumentation

```python
from ddtrace import patch_all

# Patch all supported libraries
patch_all()

# Now all database queries are automatically traced!

# SQLAlchemy
from sqlalchemy import create_engine
engine = create_engine('postgresql://localhost/mydb')
# All queries auto-traced

# Redis
import redis
r = redis.Redis()
r.get('key')  # Auto-traced

# MongoDB
from pymongo import MongoClient
client = MongoClient()
db = client.mydb
db.users.find_one({'_id': 123})  # Auto-traced
```

---

## 📱 Instrumentation: Node.js

### Installation

```bash
npm install dd-trace --save
```

### Setup

```javascript
// tracer.js - Load at app entry point
const tracer = require('dd-trace').init({
  service: 'my-nodejs-app',
  env: 'production',
  version: '1.2.3',
  hostname: 'localhost',
  port: 8126,
  
  // Automatic instrumentation
  plugins: true,
  
  // Log injection
  logInjection: true,
  
  // Sampling
  sampleRate: 1.0,  // 100%
  
  // Analytics
  analytics: true
});

module.exports = tracer;
```

```javascript
// app.js
require('./tracer');  // Must be first!

const express = require('express');
const app = express();

app.get('/users/:id', async (req, res) => {
  // Automatically traced!
  const user = await User.findById(req.params.id);
  res.json(user);
});

// Custom span
const tracer = require('dd-trace');

app.post('/orders', async (req, res) => {
  const span = tracer.startSpan('order.create');
  
  try {
    // Add tags
    span.setTag('order.items', req.body.items.length);
    span.setTag('user.id', req.user.id);
    
    // Business logic
    const order = await createOrder(req.body);
    
    span.setTag('order.id', order.id);
    res.json(order);
  } catch (error) {
    span.setTag('error', true);
    span.setTag('error.message', error.message);
    throw error;
  } finally {
    span.finish();
  }
});
```

### Express Middleware

```javascript
const tracer = require('./tracer');

// Custom middleware
app.use((req, res, next) => {
  const span = tracer.scope().active();
  
  // Add request metadata
  if (span) {
    span.setTag('user.id', req.user?.id);
    span.setTag('user.tier', req.user?.tier);
    span.setTag('request.size', req.headers['content-length']);
  }
  
  next();
});
```

---

## ☕ Instrumentation: Java

### Installation (Maven)

```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.datadoghq</groupId>
    <artifactId>dd-trace-api</artifactId>
    <version>1.x.x</version>
</dependency>
```

### Run with Agent

```bash
java -javaagent:/path/to/dd-java-agent.jar \
     -Ddd.service=my-java-app \
     -Ddd.env=production \
     -Ddd.version=1.2.3 \
     -Ddd.trace.sample.rate=1.0 \
     -jar myapp.jar
```

### Spring Boot Example

```java
import datadog.trace.api.Trace;
import datadog.trace.api.DDTags;
import io.opentracing.Span;
import io.opentracing.util.GlobalTracer;

@RestController
public class OrderController {
    
    // Automatic tracing with annotation
    @Trace(operationName = "order.create", resourceName = "POST /orders")
    @PostMapping("/orders")
    public Order createOrder(@RequestBody OrderRequest request) {
        // Automatically traced!
        
        // Get active span and add tags
        Span span = GlobalTracer.get().activeSpan();
        if (span != null) {
            span.setTag("order.items", request.getItems().size());
            span.setTag("user.id", request.getUserId());
        }
        
        Order order = orderService.create(request);
        
        if (span != null) {
            span.setTag("order.id", order.getId());
        }
        
        return order;
    }
    
    // Manual span creation
    public void processPayment(Order order) {
        Span span = GlobalTracer.get()
            .buildSpan("payment.process")
            .withTag(DDTags.SERVICE_NAME, "payment-service")
            .withTag(DDTags.RESOURCE_NAME, "stripe.charge")
            .start();
        
        try {
            span.setTag("payment.amount", order.getTotal());
            
            PaymentResult result = stripeClient.charge(order);
            
            span.setTag("payment.id", result.getId());
        } catch (Exception e) {
            span.setTag(DDTags.ERROR, true);
            span.setTag(DDTags.ERROR_MSG, e.getMessage());
            span.setTag(DDTags.ERROR_TYPE, e.getClass().getName());
            throw e;
        } finally {
            span.finish();
        }
    }
}
```

---

## 🔗 Distributed Tracing

### Context Propagation

**Problem:**
```
Request travels:
Frontend → API → Database
          ↓
      Payment Service → Stripe

How does Payment Service know it's part of same request?
→ Trace context propagation!
```

**Solution:**
```
Frontend adds headers to request:
  x-datadog-trace-id: 1234567890
  x-datadog-parent-id: 9876543210
  x-datadog-sampling-priority: 1

All downstream services read & forward these headers
→ All spans linked to same trace
```

### HTTP Headers

**Datadog Format:**
```
x-datadog-trace-id: <trace_id>
x-datadog-parent-id: <span_id>
x-datadog-sampling-priority: <0 or 1>
x-datadog-origin: <origin_name>
```

**W3C Trace Context (Standard):**
```
traceparent: 00-<trace_id>-<span_id>-<flags>
tracestate: dd=p:<parent_id>;s:<sampling>
```

### Manual Propagation

```python
# Python: Extract context from incoming request
from ddtrace.propagation.http import HTTPPropagator

def handle_request(request):
    # Extract trace context from headers
    context = HTTPPropagator.extract(request.headers)
    
    # Create child span
    with tracer.trace('my.operation', child_of=context):
        result = do_work()
    
    return result

# Inject context into outgoing request
import requests
from ddtrace.propagation.http import HTTPPropagator

def call_external_service():
    headers = {}
    HTTPPropagator.inject(tracer.current_span().context, headers)
    
    response = requests.get(
        'http://external-service/api',
        headers=headers
    )
    return response
```

```javascript
// Node.js: Automatic with supported libraries
const axios = require('axios');

// Trace context automatically propagated!
const response = await axios.get('http://other-service/api');
```

---

## 🔍 Analyzing Traces

### Finding Performance Issues

**1. Slow Traces**
```
APM → Traces → Filter:
  Duration > 1s
  Status: All
  Time: Last 4 hours

Sort by: Duration (Desc)
→ Find slowest requests
→ Click to see flame graph
→ Identify bottleneck
```

**2. Error Traces**
```
APM → Traces → Filter:
  Status: Error
  Service: payment-service
  Time: Last 1 hour

→ See all failed requests
→ Click to see error details
→ Find root cause
```

**3. Service Analytics**
```
APM → Services → Select service
→ See:
  - Request throughput
  - Latency (p50, p75, p95, p99)
  - Error rate %
  - Apdex score
```

### Key Metrics

**Latency Percentiles:**
```
p50 (median): 100ms  - Half of requests faster
p75: 200ms           - 75% faster
p95: 500ms           - 95% faster, good SLA target
p99: 1000ms          - 99% faster, catch outliers
```

**Apdex Score:**
```
Application Performance Index
0.0 - 1.0 scale

Calculation:
Satisfied requests (< T) + (Tolerating (< 4T) / 2)
─────────────────────────────────────────────────
            Total requests

Example:
T = 500ms target
100 requests:
- 80 < 500ms (Satisfied)
- 15 < 2s (Tolerating)
- 5 > 2s (Frustrated)

Apdex = (80 + 15/2) / 100 = 0.875

Interpretation:
0.94 - 1.00: Excellent
0.85 - 0.94: Good
0.70 - 0.85: Fair
0.50 - 0.70: Poor
0.00 - 0.50: Unacceptable
```

---

## 🎯 Optimization Strategies

### 1. Identify N+1 Queries

**Problem:**
```
GET /users/123/orders

Code:
  user = User.get(123)              # 1 query
  for order in user.orders:         # N queries!
      print(order.product.name)     # N more queries!

Result: 1 + N + N queries = SLOW
```

**Trace shows:**
```
span: get_user (10ms)
  span: db.query SELECT * FROM users (10ms)
span: get_orders (1000ms) ⚠️
  span: db.query SELECT * FROM orders (10ms)
  span: db.query SELECT * FROM products (5ms)  ← Repeated 100 times!
  span: db.query SELECT * FROM products (5ms)
  ...
```

**Solution:**
```python
# Eager loading
user = User.query.options(
    joinedload(User.orders).joinedload(Order.product)
).get(123)

# Now: 1-2 queries instead of 200+
```

### 2. Cache Expensive Operations

```python
from ddtrace import tracer
import redis

cache = redis.Redis()

def get_user_recommendations(user_id):
    # Check trace before optimization
    with tracer.trace('recommendations.get'):
        # This was taking 500ms
        cache_key = f'recommendations:{user_id}'
        
        # Try cache
        with tracer.trace('cache.get'):
            cached = cache.get(cache_key)
            if cached:
                return json.loads(cached)
        
        # Cache miss - compute
        with tracer.trace('recommendations.compute'):
            recommendations = expensive_ml_computation(user_id)
            cache.setex(cache_key, 3600, json.dumps(recommendations))
            return recommendations

# After: 500ms → 5ms for cached requests
```

### 3. Async Processing

```python
from celery import Celery
from ddtrace import tracer

celery = Celery('tasks')

@app.route('/orders', methods=['POST'])
def create_order():
    with tracer.trace('order.create'):
        order = Order.create(request.json)
        
        # Send email asynchronously
        # Don't wait for completion
        send_email_task.delay(order.id)
        
        return jsonify(order.to_dict()), 201

@celery.task
def send_email_task(order_id):
    # Traced separately
    with tracer.trace('email.send'):
        send_confirmation_email(order_id)

# Before: 300ms (order + email)
# After: 50ms (order only, email in background)
```

---

## 📊 Custom Metrics from Traces

### Trace Metrics
```
Datadog automatically creates metrics from traces:

trace.<service>.<resource>.hits
  → Request count

trace.<service>.<resource>.duration
  → Request duration (for percentiles)

trace.<service>.<resource>.errors
  → Error count

trace.<service>.<resource>.apdex
  → Apdex score
```

### Analyzed Spans
```yaml
# Create custom metrics from any span tag

apm_config:
  analyzed_spans:
    # Format: service|operation: rate
    payment-service|stripe.charge: 1.0
    api-gateway|request: 0.5

# Creates metrics you can use in dashboards/monitors:
trace.stripe.charge.duration
trace.stripe.charge.errors
... with all span tags
```

---

## 📝 Tóm Tắt

### Key Concepts
```
1. TRACE
   - Full journey of a request
   - Unique trace_id
   - Contains multiple spans

2. SPAN
   - Single operation in trace
   - Has duration, tags, status
   - Parent-child relationships

3. SERVICE
   - Independent component
   - Has service map
   - Performance analytics

4. RESOURCE
   - Specific endpoint/action
   - Latency tracking
   - Error tracking
```

### Setup Checklist
```
✅ Enable APM in Agent
✅ Install tracer library
✅ Instrument application
✅ Add custom tags
✅ Test traces in UI
✅ Set up service map
✅ Create APM dashboards
✅ Configure monitors
```

### Optimization Workflow
```
1. Find slow traces
2. Analyze flame graph
3. Identify bottleneck
4. Optimize code
5. Deploy
6. Verify improvement
7. Repeat
```

---

## 🤔 APM Implementation Decisions

### APM setup considerations:

```
1. Enable APM in Agent
2. Install tracer for your language
3. Instrument a simple app:
   - 1 endpoint: GET /users
   - 1 database query
4. Generate traffic (10 requests)
5. View traces in Datadog UI
6. Screenshot of trace flame graph
```

### Custom instrumentation:

```python
# Add custom instrumentation

Requirements:
1. Wrap expensive function with span
2. Add 3 custom tags
3. Simulate error (try/except)
4. Mark span with error
5. View in UI with tags and error
```

### Distributed tracing design:

```
Setup 2 services:
- Service A: API (calls Service B)
- Service B: Worker (does processing)

1. Instrument both services
2. Service A makes HTTP call to Service B
3. Verify trace spans both services
4. Add custom tags in each service
5. View complete trace in UI
```

### Performance analysis framework:

```
Given trace showing:
- Total: 850ms
- DB query: 600ms ⚠️
- External API: 200ms
- Business logic: 50ms

Questions:
1. What's the bottleneck?
2. How would you optimize?
3. What's expected improvement?
4. How to verify after fix?
```

---

## ➡️ Bước Tiếp Theo

Bạn đã master APM!

**Bài tiếp theo**: [08 - Log Management](08-LOG-MANAGEMENT.md)

Trong bài tiếp theo, chúng ta sẽ học về thu thập, phân tích và correlate logs với traces.

---

**📌 Ghi Chú Của Bạn**
```
(Service names, trace IDs, optimization ideas)








```

