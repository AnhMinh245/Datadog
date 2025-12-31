# 02 - CÁC KHÁI NIỆM CƠ BẢN

## 🎯 Mục Tiêu Bài Học
Sau bài học này, bạn sẽ nắm vững:
- Các thuật ngữ quan trọng trong Datadog
- Cấu trúc dữ liệu và cách tổ chức
- Relationships giữa các concepts
- Tags và cách sử dụng

---

## 📚 Glossary - Từ Điển Thuật Ngữ

### 1. **Agent**

**Định nghĩa:**
> Chương trình chạy trên server/container để thu thập và gửi dữ liệu về Datadog

**Ví dụ thực tế:**
```
Giống như một "nhân viên thu thập thông tin" được đặt tại mỗi server:
- Đo CPU, RAM mỗi 15 giây
- Đọc log files
- Gửi về Datadog headquarters
```

**Code example - Cài đặt Agent:**
```bash
# Ubuntu/Debian
DD_AGENT_MAJOR_VERSION=7 DD_API_KEY=<YOUR_API_KEY> \
DD_SITE="datadoghq.com" bash -c "$(curl -L https://s3.amazonaws.com/dd-agent/scripts/install_script.sh)"
```

**Thành phần Agent:**
```
Datadog Agent
├── Collector (Thu thập metrics)
├── DogStatsD (Nhận custom metrics)
├── Log Agent (Thu thập logs)
└── Trace Agent (Nhận traces)
```

---

### 2. **Host**

**Định nghĩa:**
> Một máy vật lý hoặc máy ảo chạy Agent

**Ví dụ:**
```
1 EC2 instance = 1 host
1 Physical server = 1 host
1 Container ≠ 1 host (containers thuộc về host)
```

**Lưu ý quan trọng:**
- Pricing thường tính theo số lượng host
- 1 host có thể chạy nhiều containers
- 1 host có thể chạy nhiều services

---

### 3. **Metric**

**Định nghĩa:**
> Giá trị số đo lường một khía cạnh của hệ thống theo thời gian

**4 Loại Metrics:**

#### a) **Gauge** (Đồng hồ)
Giá trị tại một thời điểm cụ thể

```python
# Ví dụ: Số users đang online
users_online = 1250

# Có thể lên xuống tự do
10:00 → 1250 users
10:05 → 1180 users
10:10 → 1420 users
```

**Use cases:**
- CPU usage
- Memory usage
- Queue size
- Temperature

#### b) **Count** (Đếm)
Số lần một sự kiện xảy ra trong khoảng thời gian

```python
# Ví dụ: Số requests
requests_total = 500  # trong 1 phút

# Luôn bắt đầu từ 0 mỗi interval
Minute 1: 500 requests
Minute 2: 750 requests
Minute 3: 620 requests
```

**Use cases:**
- HTTP requests
- Errors count
- Login attempts
- Transactions

#### c) **Rate** (Tỷ lệ)
Count chia cho thời gian (per second)

```python
# Ví dụ: Requests per second
rate = count / time_interval

500 requests / 60 seconds = 8.33 requests/second
```

**Use cases:**
- Requests per second (RPS)
- Errors per minute
- Throughput

#### d) **Distribution** (Phân phối)
Đo lường phân phối thống kê của giá trị

```python
# Ví dụ: Response times
response_times = [100ms, 150ms, 200ms, 120ms, 180ms, ...]

Tính toán:
- Average: 150ms
- p50 (median): 150ms
- p95: 195ms
- p99: 198ms
- Max: 200ms
```

**Use cases:**
- Response time
- Request size
- Database query duration

---

### 4. **Tag**

**Định nghĩa:**
> Labels để phân loại và filter metrics, logs, traces

**Cú pháp:**
```
key:value
```

**Ví dụ thực tế:**
```python
# Tags cho một web server
env:production
service:web-api
version:2.5.0
region:us-east-1
team:backend
```

**Best Practices cho Tags:**

```
✅ Tốt:
env:production
service:payment-api
region:us-west-2

❌ Không tốt:
production           # Thiếu key
env:prod:server-1    # Quá nhiều cấp
server-payment-api   # Không có cấu trúc
```

**Unified Service Tagging** (Quan trọng!):
```
env:production
service:payment-api
version:v1.2.3
```
Ba tags này luôn đi cùng nhau!

---

### 5. **Dashboard**

**Định nghĩa:**
> Giao diện hiển thị trực quan metrics và logs

**3 Loại Dashboard:**

#### a) **Timeboard**
```
Đặc điểm:
- Tất cả widgets share cùng timeframe
- Tốt cho troubleshooting
- Có thể zoom timeline
```

#### b) **Screenboard**
```
Đặc điểm:
- Mỗi widget có timeframe riêng
- Layout tự do
- Tốt cho overview
```

#### c) **System Dashboard**
```
Đặc điểm:
- Built-in dashboards
- Không thể edit
- Best practice templates
```

**Widgets phổ biến:**
```
📊 Timeseries - Biểu đồ theo thời gian
📈 Query Value - Hiển thị 1 số
🔥 Heatmap - Phân bố nhiệt
📝 Log Stream - Stream logs realtime
🗺️ Topology Map - Service map
```

---

### 6. **Monitor (Alert)**

**Định nghĩa:**
> Rule tự động kiểm tra metrics và gửi alert khi có vấn đề

**4 Loại Monitors Chính:**

#### a) **Metric Monitor**
```yaml
# Ví dụ: Alert khi CPU cao
Condition: avg(cpu.usage) > 80%
Timeframe: last 5 minutes
Alert: Send to Slack #alerts
```

#### b) **APM Monitor**
```yaml
# Ví dụ: Alert khi API chậm
Condition: p95(response_time) > 1000ms
Service: payment-api
Alert: Page oncall engineer
```

#### c) **Log Monitor**
```yaml
# Ví dụ: Alert khi có ERROR logs
Condition: count(ERROR) > 10
Timeframe: last 10 minutes
Alert: Create PagerDuty incident
```

#### d) **Composite Monitor**
```yaml
# Ví dụ: Alert khi CPU cao VÀ Memory cao
Condition: (CPU > 80%) AND (Memory > 90%)
Alert: Critical incident
```

**States của Monitor:**
```
🟢 OK      - Mọi thứ bình thường
🟡 WARN    - Gần đạt threshold
🔴 ALERT   - Đã đạt threshold, cần xử lý
⚪ NO DATA - Không nhận được data
```

---

### 7. **Service**

**Định nghĩa:**
> Một thành phần độc lập của ứng dụng (thường là microservice)

**Ví dụ e-commerce system:**
```
Application: E-commerce
├── frontend-web
├── mobile-api
├── payment-service
├── inventory-service
├── notification-service
└── recommendation-engine
```

**Service trong Datadog:**
```python
# Khai báo service trong code
from ddtrace import tracer

tracer.set_service_info(
    service='payment-service',
    app='ecommerce',
    app_type='web'
)
```

---

### 8. **Trace**

**Định nghĩa:**
> Đường đi của một request qua nhiều services

**Cấu trúc Trace:**
```
Trace (toàn bộ journey)
└── Span (một bước trong journey)
    ├── Span ID
    ├── Parent Span ID
    ├── Duration
    └── Tags
```

**Ví dụ thực tế:**
```
TRACE: Checkout Order (#12345)
│
├─ SPAN: Frontend Request (50ms)
│  └─ Tags: user_id:9999, browser:chrome
│
├─ SPAN: API Gateway (20ms)
│
├─ SPAN: Payment Service (300ms) ⚠️
│  ├─ SPAN: Validate Card (50ms)
│  ├─ SPAN: Call Stripe API (200ms) ⚠️ SLOW!
│  └─ SPAN: Update Database (50ms)
│
└─ SPAN: Email Service (100ms)
   └─ SPAN: Send Receipt Email (100ms)

Total: 470ms
```

**Flame Graph:**
```
Frontend Request    [━━━━━]
  API Gateway         [━━]
    Payment Service      [━━━━━━━━━━━━━━━]
      Validate            [━━]
      Stripe API            [━━━━━━━━━━] ⚠️
      Database                [━━]
    Email Service           [━━━━━━]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
0ms              200ms              400ms
```

---

### 9. **Log**

**Định nghĩa:**
> Text records ghi lại events trong hệ thống

**Anatomy của một Log:**
```json
{
  "timestamp": "2026-01-01T10:30:15.234Z",
  "status": "error",
  "service": "payment-api",
  "message": "Payment processing failed",
  "error": {
    "type": "StripeException",
    "message": "Card declined"
  },
  "user_id": "user_12345",
  "trace_id": "1234567890abcdef",
  "span_id": "abcdef1234567890"
}
```

**Log Levels:**
```python
DEBUG    # Chi tiết cho developer
INFO     # Thông tin chung
WARN     # Cảnh báo, không critical
ERROR    # Lỗi cần xử lý
CRITICAL # Lỗi nghiêm trọng, hệ thống có thể down
```

**Log Attributes:**
- **Standard Attributes**: timestamp, status, service, message
- **Custom Attributes**: user_id, order_id, amount, etc.
- **Reserved Attributes**: host, source, tags

---

### 10. **Integration**

**Định nghĩa:**
> Kết nối Datadog với các services/tools khác

**3 Loại Integrations:**

#### a) **Agent-based Integrations**
```
Cài đặt qua Agent config:
- PostgreSQL
- Redis
- Nginx
- Docker
```

#### b) **API Integrations**
```
Connect qua API credentials:
- AWS
- Google Cloud
- Azure
- Slack
```

#### c) **Library Integrations**
```
Import trong code:
- Python
- Node.js
- Java
- Ruby
```

**Ví dụ - PostgreSQL Integration:**
```yaml
# /etc/datadog-agent/conf.d/postgres.d/conf.yaml
init_config:

instances:
  - host: localhost
    port: 5432
    username: datadog
    password: <PASSWORD>
    dbname: production
    tags:
      - env:production
      - db:main
```

---

## 🔗 Relationships - Mối Quan Hệ

### Data Model của Datadog

```
Organization (Công ty)
│
├── Account (Team/Department)
│   │
│   ├── Host 1
│   │   ├── Agent
│   │   ├── Services
│   │   │   ├── service-a
│   │   │   └── service-b
│   │   └── Containers
│   │       ├── container-1
│   │       └── container-2
│   │
│   ├── Host 2
│   │   └── ...
│   │
│   ├── Metrics
│   │   └── Tagged by: host, service, env, etc.
│   │
│   ├── Logs
│   │   └── Linked to: traces, metrics
│   │
│   └── Traces
│       └── Linked to: logs, metrics
```

### Correlation - Liên Kết Dữ Liệu

**Tại sao cần correlation?**
```
Scenario: User báo payment failed

Không có correlation:
❌ Xem metrics → Không thấy vấn đề
❌ Xem logs → Tìm mãi không ra
❌ Xem traces → Không biết log nào liên quan

Có correlation:
✅ Click vào trace của request
✅ Tự động show logs liên quan
✅ Thấy metrics của service trong khoảng thời gian đó
✅ Root cause: Database connection timeout
```

**Cách Datadog link data:**
```python
# Trong code, inject trace context vào logs
import logging
from ddtrace import tracer

logger = logging.getLogger(__name__)

# Datadog tự động link trace_id và span_id vào logs
logger.info(
    "Payment processed",
    extra={
        'dd.trace_id': tracer.current_span().trace_id,
        'dd.span_id': tracer.current_span().span_id
    }
)
```

---

## 🏷️ Tags Deep Dive

### Tag Sources

**1. Agent Config Tags**
```yaml
# datadog.yaml
tags:
  - env:production
  - datacenter:us1
  - role:database
```

**2. Integration Tags**
```yaml
# Tự động từ AWS integration
aws.instance-type:t3.large
aws.availability-zone:us-east-1a
region:us-east-1
```

**3. Application Tags**
```python
# Trong code
from ddtrace import tracer

@tracer.wrap(service='payment', resource='process_payment')
def process_payment(amount):
    span = tracer.current_span()
    span.set_tag('payment.amount', amount)
    span.set_tag('payment.method', 'credit_card')
```

**4. Custom Tags**
```python
# DogStatsD
from datadog import statsd

statsd.increment(
    'payment.success',
    tags=['env:prod', 'currency:usd', 'country:us']
)
```

### Tag Best Practices

**Reserved Tags (Luôn dùng):**
```
env:production
service:payment-api
version:v1.2.3
```

**Recommended Tags:**
```
# Infrastructure
host:web-server-01
region:us-east-1
availability-zone:us-east-1a
instance-type:t3.large

# Application
service:payment-api
team:payments
component:backend
language:python

# Business
customer-tier:premium
feature-flag:new-checkout
ab-test:variant-b
```

**Tag Cardinality (Quan trọng!):**

```
❌ HIGH CARDINALITY (Tránh!):
user-id:12345
request-id:abc123
timestamp:1234567890

Lý do: Tạo ra quá nhiều unique combinations
→ Tốn tiền
→ Slow queries
→ Datadog có thể block

✅ LOW CARDINALITY (Tốt):
env:production
service:api
region:us-east-1
team:backend

Lý do: Giới hạn số values
→ Efficient queries
→ Cost-effective
```

### Tag Queries

**Filter metrics bằng tags:**
```
# Syntax
metric{tag_key:tag_value}

# Examples
avg:system.cpu.user{env:production}
sum:http.requests{service:web-api,status:200}
max:database.connections{env:prod,region:us-east-1}
```

**Aggregation với tags:**
```
# Group by tag
avg:system.cpu.user{*} by {host}
sum:http.requests{*} by {service,status}
```

---

## 📊 Data Types Summary

### Quick Reference Table

| Data Type | Update Frequency | Retention | Use Case |
|-----------|-----------------|-----------|----------|
| **Metrics** | 15 seconds | 15 months | Performance monitoring |
| **Logs** | Real-time | Custom (3-30 days) | Debugging, audit |
| **Traces** | Real-time | 15 days | Request flow, latency |
| **Events** | On occurrence | 1 year | Deploys, incidents |
| **Synthetics** | 1-60 minutes | 1 year | Proactive monitoring |

### Storage & Indexing

```
┌─────────────────────────────────────┐
│ Ingestion (Data vào Datadog)       │
│ - Tất cả data được ingest           │
│ - Có thể filter ở Agent level      │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ Indexing (Lưu trữ để query)        │
│ - Chỉ index % data (tùy config)   │
│ - Đây là phần tính tiền chính      │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ Archives (Lưu trữ dài hạn)         │
│ - Gửi tất cả vào S3/GCS            │
│ - Không query được trực tiếp       │
│ - Dùng cho compliance              │
└─────────────────────────────────────┘
```

---

## 🎓 Concepts Map

### Tổng Quan Mối Quan Hệ

```
┌──────────────────────────────────────────────────┐
│                 DATADOG PLATFORM                 │
├──────────────────────────────────────────────────┤
│                                                  │
│  ┌─────────────┐      ┌──────────────┐         │
│  │   AGENT     │─────▶│  DATADOG     │         │
│  │  (Collect)  │      │   BACKEND    │         │
│  └─────────────┘      └──────┬───────┘         │
│         │                     │                  │
│         │                     ▼                  │
│    ┌────┴────┐    ┌──────────────────┐         │
│    │ Metrics │    │   DASHBOARDS     │         │
│    │  Logs   │    │   MONITORS       │         │
│    │ Traces  │    │   ALERTS         │         │
│    └─────────┘    └──────────────────┘         │
│         │                     │                  │
│         │                     ▼                  │
│         └────────────▶┌──────────────┐         │
│                       │ INTEGRATIONS │         │
│                       │   Slack      │         │
│                       │   PagerDuty  │         │
│                       │   Jira       │         │
│                       └──────────────┘         │
└──────────────────────────────────────────────────┘
```

---

## 📝 Tóm Tắt

### Must-Know Concepts

1. **Agent**: Collector chạy trên host
2. **Metric**: Số liệu theo thời gian (4 types: gauge, count, rate, distribution)
3. **Tag**: key:value labels để filter và group
4. **Service**: Independent component của app
5. **Trace**: Journey của request qua services
6. **Log**: Text records của events
7. **Dashboard**: Visual interface
8. **Monitor**: Alert rules

### Key Principles

```
✅ Tags are everything - Tag đúng, query dễ
✅ Correlation is power - Link metrics, logs, traces
✅ Low cardinality - Tránh tags có quá nhiều unique values
✅ Unified Service Tagging - env, service, version luôn đi cùng nhau
```

---

## 🎯 Bài Tập

### Exercise 1: Terminology Quiz

Điền định nghĩa:
```
1. Agent là: _____
2. 4 loại metrics: _____
3. Trace khác Log như thế nào: _____
4. Tag cardinality là gì: _____
```

### Exercise 2: Tagging Strategy

Thiết kế tags cho hệ thống:
```
System: E-commerce website
- 3 environments: dev, staging, production
- 5 services: web, api, payment, inventory, notification
- 3 regions: us-east, us-west, eu-west

Tạo tagging strategy:
env: _____
service: _____
region: _____
...
```

### Exercise 3: Data Type Selection

Chọn đúng data type:
```
1. Theo dõi số users online → _____
2. Debug lỗi thanh toán → _____
3. Tìm service chậm nhất → _____
4. Alert khi CPU cao → _____
```

---

## ➡️ Bước Tiếp Theo

Bây giờ bạn đã nắm vững các khái niệm cơ bản!

**Bài tiếp theo**: [03 - Kiến Trúc Datadog](03-KIEN-TRUC-DATADOG.md)

Trong bài tiếp theo, chúng ta sẽ tìm hiểu sâu hơn về kiến trúc và cách Datadog hoạt động.

---

**📌 Ghi Chú Của Bạn**
```
(Viết ghi chú, câu hỏi, hoặc ý tưởng của bạn ở đây)








```

