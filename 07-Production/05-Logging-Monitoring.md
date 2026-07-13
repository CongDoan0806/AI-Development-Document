# Logging & Monitoring

## 1. Mục Tiêu

Theo dõi production AI app để:
- 🐛 **Debug**: trace error nhanh
- 📊 **Monitor**: latency, cost, error rate
- 📈 **Improve**: identify slow/expensive queries
- 🚨 **Alert**: when something breaks
- 💼 **Audit**: who did what, when

---

## 2. Structured Logging

### 2.1 Setup

```python
import logging
import json
import sys

class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_data = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
        }
        # Add extra fields
        if hasattr(record, "extra"):
            log_data.update(record.extra)
        return json.dumps(log_data)

handler = logging.StreamHandler(sys.stdout)
handler.setFormatter(JSONFormatter())

logger = logging.getLogger("my-app")
logger.addHandler(handler)
logger.setLevel(logging.INFO)
```

### 2.2 Usage

```python
logger.info("Query received", extra={
    "user_id": "u123",
    "question_length": 50,
    "session_id": "s456",
})
```

Output:
```json
{
    "timestamp": "2025-01-15T10:30:00",
    "level": "INFO",
    "logger": "my-app",
    "message": "Query received",
    "user_id": "u123",
    "question_length": 50,
    "session_id": "s456"
}
```

---

## 3. LangSmith Production Setup

Mọi LLM/chain/tool call tự log lên LangSmith:

```env
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=lsv2_...
LANGCHAIN_PROJECT=production
```

→ Đã học chi tiết ở [Phase 6 - LangSmith](../06-Evaluation/04-LangSmith.md).

---

## 4. Custom Callbacks

```python
from langchain_core.callbacks import BaseCallbackHandler

class ProductionCallback(BaseCallbackHandler):
    def __init__(self, logger, metrics):
        self.logger = logger
        self.metrics = metrics
        self.start_time = None
    
    def on_chain_start(self, serialized, inputs, **kwargs):
        self.start_time = time.time()
        self.logger.info("Chain started", extra={
            "chain": serialized.get("name"),
            "input_size": len(str(inputs)),
        })
    
    def on_chain_end(self, outputs, **kwargs):
        elapsed = time.time() - self.start_time
        self.logger.info("Chain ended", extra={
            "elapsed_seconds": elapsed,
            "output_size": len(str(outputs)),
        })
        self.metrics.histogram("chain_duration", elapsed)
    
    def on_llm_end(self, response, **kwargs):
        if hasattr(response, "llm_output") and response.llm_output:
            usage = response.llm_output.get("token_usage", {})
            self.metrics.counter("llm_tokens_in", usage.get("prompt_tokens", 0))
            self.metrics.counter("llm_tokens_out", usage.get("completion_tokens", 0))
    
    def on_tool_start(self, serialized, input_str, **kwargs):
        self.logger.info("Tool called", extra={
            "tool": serialized["name"],
            "input": input_str[:200],
        })
    
    def on_tool_error(self, error, **kwargs):
        self.logger.error("Tool error", extra={
            "error": str(error),
        })
        self.metrics.counter("tool_errors", 1)

# Use
result = chain.invoke(input, config={"callbacks": [ProductionCallback(logger, metrics)]})
```

---

## 5. Prometheus Metrics

```python
from prometheus_client import Counter, Histogram, Gauge

REQUEST_COUNT = Counter(
    "ai_requests_total",
    "Total AI requests",
    labelnames=["endpoint", "status"],
)

LATENCY = Histogram(
    "ai_request_duration_seconds",
    "Request latency",
    labelnames=["endpoint"],
    buckets=(0.1, 0.5, 1, 2, 5, 10, 30),
)

TOKEN_USAGE = Counter(
    "ai_tokens_total",
    "Tokens used",
    labelnames=["model", "type"],
)

ACTIVE_REQUESTS = Gauge(
    "ai_active_requests",
    "Currently active requests",
)

# In endpoint
@app.post("/query")
async def query(req):
    ACTIVE_REQUESTS.inc()
    start = time.time()
    
    try:
        result = await chain.ainvoke(req.question)
        REQUEST_COUNT.labels(endpoint="query", status="success").inc()
        return result
    except Exception as e:
        REQUEST_COUNT.labels(endpoint="query", status="error").inc()
        raise
    finally:
        LATENCY.labels(endpoint="query").observe(time.time() - start)
        ACTIVE_REQUESTS.dec()
```

### Expose Metrics

```python
from prometheus_fastapi_instrumentator import Instrumentator

Instrumentator().instrument(app).expose(app, endpoint="/metrics")
```

---

## 6. Grafana Dashboards

Sample queries:

```promql
# Request rate
rate(ai_requests_total[5m])

# Error rate
rate(ai_requests_total{status="error"}[5m]) 
  / rate(ai_requests_total[5m])

# p95 latency
histogram_quantile(0.95, rate(ai_request_duration_seconds_bucket[5m]))

# Token cost per hour
sum(rate(ai_tokens_total{type="input"}[1h])) * 3600 * 0.0000015
  + sum(rate(ai_tokens_total{type="output"}[1h])) * 3600 * 0.000006
```

---

## 7. OpenTelemetry

Chuẩn distributed tracing:

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

# Setup
trace.set_tracer_provider(TracerProvider())
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(OTLPSpanExporter(endpoint="http://jaeger:4317"))
)
tracer = trace.get_tracer(__name__)

# Auto-instrument FastAPI
FastAPIInstrumentor.instrument_app(app)

# Manual spans
@app.post("/query")
async def query(req):
    with tracer.start_as_current_span("query_endpoint") as span:
        span.set_attribute("user_id", req.user_id)
        
        with tracer.start_as_current_span("retrieve"):
            docs = await retriever.ainvoke(req.question)
        
        with tracer.start_as_current_span("generate"):
            answer = await chain.ainvoke(...)
        
        return answer
```

---

## 8. Log Aggregation

### Datadog

```python
# pip install ddtrace
from ddtrace import patch_all
patch_all()  # Auto-instrument

# Custom traces
from ddtrace import tracer

@tracer.wrap()
def query(req):
    ...
```

### CloudWatch (AWS)

```python
import watchtower

handler = watchtower.CloudWatchLogHandler(log_group="my-ai-app")
logger.addHandler(handler)
```

### ELK (Elasticsearch, Logstash, Kibana)

```python
# Logs đi đến Logstash via syslog
import logging.handlers

handler = logging.handlers.SysLogHandler(address=("logstash", 5000))
logger.addHandler(handler)
```

---

## 9. Audit Logs

```python
class AuditLog:
    def __init__(self, db):
        self.db = db
    
    def log_query(self, user_id, question, answer, sources):
        self.db.insert({
            "type": "query",
            "user_id": user_id,
            "question": question,
            "answer": answer,
            "sources": sources,
            "timestamp": datetime.now(),
            "ip": get_client_ip(),
        })
    
    def log_admin_action(self, user_id, action, details):
        self.db.insert({
            "type": "admin",
            "user_id": user_id,
            "action": action,
            "details": details,
            "timestamp": datetime.now(),
        })

# Audit all queries
audit = AuditLog(db)

@app.post("/query")
async def query(req, user=Depends(get_user)):
    result = await chain.ainvoke(req.question)
    audit.log_query(user.id, req.question, result["answer"], result["sources"])
    return result
```

---

## 10. Error Tracking - Sentry

```python
# pip install sentry-sdk
import sentry_sdk
from sentry_sdk.integrations.fastapi import FastApiIntegration

sentry_sdk.init(
    dsn="https://...@sentry.io/...",
    integrations=[FastApiIntegration()],
    traces_sample_rate=0.1,
    environment="production",
)

# Auto-capture errors
@app.post("/query")
async def query(req):
    try:
        return await chain.ainvoke(req.question)
    except Exception as e:
        sentry_sdk.capture_exception(e)
        raise
```

---

## 11. Alerts

### 11.1 Threshold-Based

```python
# Cron job check daily
def check_alerts():
    # Error rate
    error_rate = get_error_rate_last_hour()
    if error_rate > 0.05:  # 5%
        send_alert(f"Error rate high: {error_rate:.1%}")
    
    # Latency
    p95 = get_p95_latency()
    if p95 > 10:
        send_alert(f"p95 latency: {p95}s")
    
    # Cost
    daily_cost = get_daily_cost()
    if daily_cost > 100:
        send_alert(f"Daily cost: ${daily_cost}")

def send_alert(message):
    # Slack webhook
    requests.post(SLACK_WEBHOOK, json={"text": message})
    # Or PagerDuty, email, ...
```

### 11.2 Anomaly Detection

```python
import numpy as np

def detect_anomaly(metric_values, threshold_sigma=3):
    mean = np.mean(metric_values[-100:])
    std = np.std(metric_values[-100:])
    
    current = metric_values[-1]
    if abs(current - mean) > threshold_sigma * std:
        return True
    return False
```

---

## 12. Production-Ready Logger Class

```python
import logging
import json
import time
import uuid
from contextvars import ContextVar

request_id_var: ContextVar[str] = ContextVar("request_id", default="")
user_id_var: ContextVar[str] = ContextVar("user_id", default="")

class ContextFilter(logging.Filter):
    def filter(self, record):
        record.request_id = request_id_var.get()
        record.user_id = user_id_var.get()
        return True

class JSONFormatter(logging.Formatter):
    def format(self, record):
        log = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
            "request_id": getattr(record, "request_id", ""),
            "user_id": getattr(record, "user_id", ""),
        }
        if record.exc_info:
            log["exception"] = self.formatException(record.exc_info)
        if hasattr(record, "extra_data"):
            log.update(record.extra_data)
        return json.dumps(log, default=str)

def setup_logging():
    handler = logging.StreamHandler()
    handler.setFormatter(JSONFormatter())
    handler.addFilter(ContextFilter())
    
    root = logging.getLogger()
    root.handlers = [handler]
    root.setLevel(logging.INFO)

# In middleware
@app.middleware("http")
async def add_request_context(request, call_next):
    request_id_var.set(str(uuid.uuid4()))
    response = await call_next(request)
    return response
```

---

## 13. Best Practices

✅ **Structured logging** (JSON)

✅ **Context propagation**: request_id, user_id everywhere

✅ **Sampling**: trace 10% requests cho high-volume

✅ **Separate concerns**: app logs, audit logs, traces, metrics

✅ **Retention policy**: keep N days

✅ **PII redaction**: don't log sensitive data

✅ **Alert on SLI/SLO**: not random metrics

---

## 14. Checklist

- [ ] Structured JSON logging
- [ ] LangSmith production tracing
- [ ] Custom callbacks
- [ ] Prometheus metrics
- [ ] OpenTelemetry traces
- [ ] Sentry error tracking
- [ ] Audit logs
- [ ] Alerting

➡️ **Tiếp theo**: [Scaling](./06-Scaling.md)
