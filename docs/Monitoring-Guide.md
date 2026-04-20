---
layout: default
title: Monitoring Guide
nav_order: 24
---

<details markdown="block">
  <summary>
    Table of Contents
  </summary>
  {: .text-delta }
- TOC
{:toc}
</details>

<span class="big-text">Monitoring Guide</span><br/><span class="med-text">Health Checks, Metrics, and Observability</span>

---

# Overview

SWIRL provides health check and metrics endpoints for comprehensive monitoring in production deployments. These endpoints enable integration with standard monitoring tools like Kubernetes, Prometheus, and alerting systems, allowing operators to track system health, detect issues, and maintain service reliability.

# Health Check Endpoints

SWIRL exposes health check endpoints that report the status of critical system components, suitable for use as Kubernetes readiness and liveness probes.

## `/health/celery/` Endpoint

The `/health/celery/` endpoint returns the health status of Celery workers, including queue connectivity and worker availability.

### Usage

To check the health of Celery workers:

```bash
curl http://localhost:8000/health/celery/
```

### Example Response

A healthy response returns HTTP 200:

```json
{
  "status": "healthy",
  "workers": {
    "celery@worker-1": {
      "status": "online",
      "active_tasks": 2,
      "processed": 1450
    },
    "celery@worker-2": {
      "status": "online",
      "active_tasks": 1,
      "processed": 1320
    }
  },
  "queues": {
    "default": 5,
    "search": 12,
    "rag": 3
  }
}
```

### Kubernetes Configuration

Use this endpoint as a readiness probe to ensure traffic is only routed to healthy SWIRL instances:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: swirl-api
spec:
  containers:
  - name: swirl
    image: swirl-search:latest
    livenessProbe:
      httpGet:
        path: /health/celery/
        port: 8000
      initialDelaySeconds: 30
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /health/celery/
        port: 8000
      initialDelaySeconds: 10
      periodSeconds: 5
```

# Metrics Endpoints

SWIRL exposes Prometheus-compatible metrics endpoints for detailed monitoring and alerting integration.

## `/metrics/celery/` Endpoint

The `/metrics/celery/` endpoint exposes Celery queue metrics in Prometheus format, including queue depth, worker status, and task performance metrics.

### Usage

To retrieve Celery metrics:

```bash
curl http://localhost:8000/metrics/celery/
```

### Metrics Provided

The endpoint exposes metrics from the **CeleryQueueCollector**, including:

- **`celery_queue_depth`** — Number of pending tasks in each queue (default, search, rag, etc.)
- **`celery_worker_active_tasks`** — Number of currently executing tasks per worker
- **`celery_worker_processed_total`** — Total number of tasks processed by each worker
- **`celery_task_duration_seconds`** — Histogram of task execution times
- **`celery_task_errors_total`** — Count of failed tasks by task type
- **`celery_worker_heartbeat_timestamp`** — Last heartbeat time for each worker

### Example Metrics Output

```
# HELP celery_queue_depth Number of pending tasks in queue
# TYPE celery_queue_depth gauge
celery_queue_depth{queue="default"} 5
celery_queue_depth{queue="search"} 12
celery_queue_depth{queue="rag"} 3

# HELP celery_worker_active_tasks Active tasks per worker
# TYPE celery_worker_active_tasks gauge
celery_worker_active_tasks{worker="celery@worker-1"} 2
celery_worker_active_tasks{worker="celery@worker-2"} 1

# HELP celery_task_duration_seconds Task execution time
# TYPE celery_task_duration_seconds histogram
celery_task_duration_seconds_bucket{le="1.0",task="search"} 45
celery_task_duration_seconds_bucket{le="5.0",task="search"} 120
celery_task_duration_seconds_bucket{le="10.0",task="search"} 148
```

# Prometheus Integration

Integrate SWIRL with Prometheus to collect and store metrics for analysis and alerting.

## Configuration

Add the following scrape configuration to your `prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'swirl'
    static_configs:
      - targets: ['localhost:8000']
    metrics_path: '/metrics/celery/'
    scrape_interval: 30s
    scrape_timeout: 10s

  - job_name: 'swirl-django'
    static_configs:
      - targets: ['localhost:8000']
    metrics_path: '/metrics/'
    scrape_interval: 30s
```

## Verifying Prometheus Integration

1. Start Prometheus with the updated configuration:
   ```bash
   prometheus --config.file=prometheus.yml
   ```

2. Open the Prometheus UI at `http://localhost:9090`

3. Query SWIRL metrics:
   ```
   celery_queue_depth
   celery_task_duration_seconds
   celery_worker_active_tasks
   ```

# Log Files

SWIRL maintains separate log files for different components, enabling granular troubleshooting and audit trails.

## Log Locations

### `logs/django.log`

Contains Django and API server activity:
- HTTP request/response logs
- Authentication and authorization events
- API startup messages and initialization errors
- Database connection issues

**Example:**
```
[2026-04-12 10:15:30] INFO [django.request] GET /swirl/search/ 200
[2026-04-12 10:15:45] WARNING [django.db] Slow query detected: 2.3s
[2026-04-12 10:16:00] ERROR [django.security] Failed login attempt from 192.168.1.100
```

### `logs/celery-worker.log`

Contains Celery worker activity (single worker deployments):
- Search federation execution
- Result processing and aggregation
- Task processing errors
- Worker startup and shutdown events

**Enterprise Edition:** Per-worker logs in `logs/celery-worker-*.log` for multiple workers.

**Example:**
```
[2026-04-12 10:15:32] INFO [celery.tasks] Task search_federated[id=abc123] started
[2026-04-12 10:15:45] WARNING [celery.search] Provider timeout: GoogleBooks (5.2s)
[2026-04-12 10:15:50] INFO [celery.tasks] Task search_federated[id=abc123] completed (5.8s)
```

### `logs/celery-beat.log`

Contains scheduled task execution logs:
- Subscription service runs
- License expiration checks
- Data cleanup and maintenance tasks
- Search result expiration

**Example:**
```
[2026-04-12 00:05:00] INFO [celery.beat] Running scheduled task: cleanup_expired_results
[2026-04-12 12:00:00] INFO [celery.beat] Running scheduled task: check_subscriptions
[2026-04-12 06:00:00] WARNING [celery.beat] License expires in 7 days
```

## Viewing Live Logs

Use the SWIRL CLI to tail all service logs in real-time:

```bash
python swirl.py logs
```

This command displays a live stream of logs from Django, Celery workers, and Celery Beat, making it easy to monitor system activity as it happens.

To view a specific log file:

```bash
tail -f logs/django.log
tail -f logs/celery-worker.log
tail -f logs/celery-beat.log
```

# Key Metrics to Monitor

Monitor these critical metrics to ensure SWIRL operates reliably and efficiently.

## Search Latency

**What to monitor:** Time from search creation (NEW_SEARCH) to completion (FULL_RESULTS_READY)

- **Healthy:** < 10 seconds for most searches
- **Warning:** 10-20 seconds (potential provider delays or network issues)
- **Critical:** > 20 seconds (investigate provider response times or queue congestion)

**Query:**
```
celery_task_duration_seconds{task="search_federated"}
```

## Celery Queue Depth

**What to monitor:** Number of pending tasks in each queue

- **Healthy:** 0-10 tasks per queue
- **Warning:** 10-50 tasks (workers may be falling behind)
- **Critical:** > 50 tasks (worker shortage or performance degradation)

**Query:**
```
celery_queue_depth
```

## Worker Memory Consumption

Monitor memory usage per worker to detect memory leaks and prevent out-of-memory crashes.

- **Healthy:** < 2 GB per worker
- **Warning:** 2-4 GB (check for memory leaks)
- **Critical:** > 4 GB (restart worker, investigate leaks)

**Collection:** Use system monitoring tools (Prometheus `node_exporter`) to scrape worker process metrics.

## SearchProvider Response Times

Monitor response times and error rates for each data source:

- **Healthy:** < 5 seconds per provider
- **Warning:** 5-10 seconds
- **Critical:** > 10 seconds or error rate > 5%

**Metrics to track:**
```
celery_task_duration_seconds{provider="GoogleBooks"}
celery_task_errors_total{provider="GoogleBooks"}
```

## RAG Pipeline Duration

Monitor the time required to generate AI insights:

- **Healthy:** < 5 seconds
- **Warning:** 5-15 seconds
- **Critical:** > 15 seconds (check LLM provider availability)

**Query:**
```
celery_task_duration_seconds{task="rag_generate_insight"}
```

# Rate Limiting

SWIRL implements API rate limiting to protect the system from overload and ensure fair resource allocation.

## Configuration

SWIRL uses the **`DocsServiceUserIpThrottle`** throttle class for rate limiting based on user identity and IP address.

### Chat Endpoint Rate Limiting

The default chat endpoint throttle rate is **15 requests per minute**:

```python
# settings.py
DOCS_CHAT_THROTTLE_RATE = "15/minute"
```

Adjust this setting to match your deployment requirements:

```python
# Increase to 30 requests per minute
DOCS_CHAT_THROTTLE_RATE = "30/minute"

# Or disable for internal deployments (not recommended)
DOCS_CHAT_THROTTLE_RATE = None
```

### Per-SearchProvider Rate Limiting

Configure rate limits for individual search providers in the SearchProvider's `query_mappings` configuration:

```json
{
  "name": "GoogleBooks",
  "query_mappings": {
    "rate_limit": "100/hour",
    "timeout": 5,
    "retry_count": 3
  }
}
```

**Common provider rate limits:**
- **Google Books API:** 100 requests/day (configure caching)
- **Wikipedia API:** 200 requests/second
- **ArXiv API:** 3 requests/second (with delays)
- **Custom APIs:** Configure based on provider SLA

### Monitoring Rate Limit Hits

Monitor API throttling events in logs and metrics:

```bash
grep "Throttled" logs/django.log
```

**Prometheus query:**
```
rate(django_http_responses_total{status="429"}[5m])
```

If throttling is frequent, increase the throttle rate or add capacity.

# Alerting Recommendations

Configure alerts to proactively notify operations teams of potential issues.

## Recommended Alert Thresholds

### Disk Usage

```yaml
alert:
  - name: HighDiskUsage
    condition: disk_usage_percent > 80
    severity: warning
    action: Review logs and clean up old results

  - name: CriticalDiskUsage
    condition: disk_usage_percent > 95
    severity: critical
    action: Immediately clean up results or expand storage
```

### Celery Queue Depth

```yaml
alert:
  - name: HighQueueDepth
    condition: celery_queue_depth > 50
    severity: warning
    action: Scale up worker count or investigate performance

  - name: QueueBacklog
    condition: celery_queue_depth > 200
    severity: critical
    action: Immediately scale workers or pause new searches
```

### Worker Health

```yaml
alert:
  - name: WorkerOffline
    condition: celery_worker_heartbeat_timestamp > 60 seconds ago
    severity: critical
    action: Restart worker, check logs for errors

  - name: HighWorkerMemory
    condition: worker_memory_mb > 4096
    severity: warning
    action: Investigate memory leak, restart worker if persistent
```

### License Expiration

```yaml
alert:
  - name: LicenseExpiringSoon
    condition: days_until_expiration < 30
    severity: warning
    action: Renew license before expiration

  - name: LicenseExpired
    condition: current_date > license_expiration_date
    severity: critical
    action: Renew license immediately to restore functionality
```

### Provider Availability

```yaml
alert:
  - name: ProviderHighErrorRate
    condition: provider_error_rate > 5%
    severity: warning
    action: Check provider status page, verify credentials

  - name: ProviderTimeout
    condition: provider_response_time > 10 seconds
    severity: warning
    action: Check network connectivity, adjust timeout settings
```

### Search Latency

```yaml
alert:
  - name: SearchLatencyHigh
    condition: search_duration_p95 > 20 seconds
    severity: warning
    action: Review provider performance, check queue depth
```

## Setting Up Alerts in Prometheus

Add alerting rules to `prometheus-rules.yml`:

```yaml
groups:
  - name: swirl_alerts
    interval: 30s
    rules:
      - alert: HighCeleryQueueDepth
        expr: celery_queue_depth > 50
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High Celery queue depth detected"
          description: "Queue {% raw %}{{ $labels.queue }}{% endraw %} has {% raw %}{{ $value }}{% endraw %} pending tasks"

      - alert: WorkerMemoryHigh
        expr: process_resident_memory_bytes > 4294967296
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Worker memory usage exceeding 4GB"
          description: "Worker {% raw %}{{ $labels.worker }}{% endraw %} is using {% raw %}{{ $value }}{% endraw %} bytes"
```

Configure notifications via Alertmanager to email, Slack, or PagerDuty:

```yaml
# alertmanager.yml
route:
  receiver: 'default'
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h

receivers:
  - name: 'default'
    slack_configs:
      - api_url: 'YOUR_SLACK_WEBHOOK_URL'
        channel: '#alerts'
        title: 'SWIRL Alert'
```

---

## Need Help?

For additional support and questions about monitoring SWIRL:
- Review the [User Guide](./User-Guide) for system operation
- Check the [Architecture](./Architecture) documentation for system design
- Visit the [SWIRL GitHub Issues](https://github.com/swirlai/swirl-search/issues) for known issues
