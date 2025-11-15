# Custom Scheduler Architecture — FastAPI + Celery

> A complete architecture blueprint for a user-facing scheduling platform (cronhooks-like) using FastAPI, Celery, MySQL (SQLAlchemy), Redis, and React for the dashboard.

---

## Goals

- Allow users to create, edit, pause/resume, and delete schedules (cron/interval/one-off).
    
- Persist schedules and execution history to a relational DB (MySQL).
    
- Use Celery as an execution engine (task workers) — not the scheduler.
    
- Support multi-tenant use, high scale (thousands of schedules), and strong operational observability.
    
- Provide robust retry, DLQ, and audit logs for each execution.
    

## Non-goals

- Replacing full orchestration systems like Temporal or Airflow. This is a lightweight SaaS scheduler focused on webhook-style jobs and simple user-defined tasks.
    

## High-level components

1. **API Service (FastAPI)**
    
    - Public REST / GraphQL API for managing schedules, viewing logs, and controlling runs.
        
    - Authentication (Supabase / JWT / OAuth) and tenant isolation.
        
    - Exposes endpoints that the UI uses and also internal endpoints for admin operations.
        
2. **Scheduler Service (Standalone process)**
    
    - Small, horizontally-scalable process responsible for computing `due` jobs and enqueuing them into Celery.
        
    - Written in Python, uses SQLAlchemy to read/write the DB, and Redis (or DB) for lightweight locks.
        
    - Runs at a configurable tick (e.g., every 1–5 seconds) or uses an event-driven approach (see variants).
        
3. **Execution Engine — Celery Workers**
    
    - Receives tasks from the scheduler and executes them (HTTP webhooks, scripts, lambda-style code executions).
        
    - Responsible for retries, basic timeout control, and writing run results to DB.
        
4. **MySQL (Primary Store)**
    
    - Stores schedules, schedule metadata, run history (logs), tenant metadata, and user settings.
        
5. **Redis**
    
    - Broker for Celery tasks (or RabbitMQ if desired).
        
    - Optional distributed locking (Redlock) and short-lived caches.
        
6. **React Dashboard**
    
    - CRUD for schedules, run history viewer, monitoring dash, per-tenant quotas and usage, and actions (run now, pause, resume).
        
7. **Admin/Operator Tools**
    
    - Health endpoints, maintenance endpoints to re-enqueue stuck jobs, and DB/backup tools.
        
8. **Observability**
    
    - Prometheus metrics, Grafana dashboards, centralized logs (ELK / Loki), Sentry for errors, and tracing (Jaeger / OpenTelemetry).
        

---

## Data model (minimal)

### `schedules`

- `id` (UUID)
    
- `tenant_id` (UUID)
    
- `user_id` (UUID)
    
- `name` (string)
    
- `type` (`cron` | `interval` | `oneoff`)
    
- `cron_expr` (string, nullable)
    
- `interval_seconds` (int, nullable)
    
- `next_run_at` (datetime, indexed)
    
- `last_run_at` (datetime)
    
- `status` (`active` | `paused` | `deleted`)
    
- `payload` (JSON) — arbitrary meta for execution
    
- `retry_policy` (JSON) — retry attempts, backoff strategy
    
- `created_at`, `updated_at`
    

### `schedule_runs`

- `id` (UUID)
    
- `schedule_id` (UUID)
    
- `tenant_id` (UUID)
    
- `run_at` (datetime)
    
- `status` (`queued` | `running` | `success` | `failed` | `timed_out` | `dead_letter`)
    
- `worker_id` (string)
    
- `duration_ms` (int)
    
- `response` (text/json) — webhook response or execution output
    
- `error` (text)
    
- `attempt` (int)
    
- `created_at`, `updated_at`
    

### `locks` (optional)

- Used for advisory locks if you prefer DB-powered locking.
    

### Indexing

- `schedules(next_run_at, status)` — critical.
    
- `schedule_runs(schedule_id, run_at)` — for fast history queries.
    

---

## Scheduler design patterns (three variants)

### 1) Polling Scheduler (simple, reliable)

- Scheduler wakes every `tick` (1–5s).
    
- Query DB: `SELECT * FROM schedules WHERE status='active' AND next_run_at <= now()`.
    
- Acquire lock per schedule (row-level `SELECT ... FOR UPDATE` or Redis lock) to avoid double-enqueue.
    
- Enqueue Celery task with `schedule_id` and `payload`.
    
- Update `next_run_at` using `croniter` or `interval` calculation BEFORE commit.
    
- Insert a `schedule_runs` record with `status='queued'`.
    

**Pros:** Simple, easy to reason about, DB is the source of truth.  
**Cons:** Polling overhead at very large scale; must tune tick, batching, and lock handling.

### 2) Event-driven (DB triggers + queue)

- When a schedule is created/updated, push a message to a small "schedule-events" queue (Redis stream or Kafka).
    
- A lightweight consumer recalculates `next_run_at` and only schedules the next run as a delayed job.
    

**Pros:** Less polling, reactive, scales better.  
**Cons:** More moving parts, more complex to implement. Requires careful handling of clock drift and delayed job support.

### 3) Hybrid (recommended at scale)

- Use polling for short term (every few seconds) and event-driven for schedule changes.
    
- Polling handles crash recovery and missed runs; events reduce DB scan frequency.
    

---

## Locking strategies (avoid duplicate runs)

1. **DB row lock (`SELECT FOR UPDATE`)**
    
    - Start a transaction, select schedule FOR UPDATE, verify `next_run_at <= now()`, update `next_run_at`, commit.
        
    - Works well when scheduler instances are few and DB supports row locks.
        
2. **Advisory locks (DB-specific)**
    
    - Use `GET_LOCK()` in MySQL or advisory locks in Postgres with `pg_try_advisory_lock`.
        
3. **Redis distributed lock (Redlock)**
    
    - Good when DB cannot handle high contention or when scheduler processes are many.
        
4. **Optimistic update (compare-and-swap)**
    
    - Update if `next_run_at` equals expected value (single SQL `UPDATE ... WHERE id=? AND next_run_at=?`).
        

Choose the strategy that best matches your RPS and operational tolerance.

---

## Scheduler pseudocode (polling pattern)

```python
# run every T seconds
with db.session() as s:
    now = utcnow()
    due = s.query(Schedule).filter(
        Schedule.status=='active',
        Schedule.next_run_at <= now
    ).limit(BATCH_SIZE).all()

    for schedule in due:
        # try to claim the schedule
        claimed = try_claim(schedule.id)
        if not claimed:
            continue

        # compute next run
        schedule.next_run_at = compute_next(schedule)
        run = ScheduleRun(schedule_id=schedule.id, run_at=now, status='queued')
        s.add(run)
        s.commit()

        # enqueue to celery
        celery.send_task('tasks.execute_schedule', args=[run.id])
```

`try_claim()` may use any of the locking strategies above.

---

## Celery task behavior

```python
@app.task(bind=True, acks_late=True)
def execute_schedule(self, run_id):
    run = db.get_run(run_id)
    db.update_run_status(run_id, 'running', worker_id=self.request.hostname)

    try:
        result = perform_webhook_or_action(run.schedule.payload)
        db.update_run(run_id, status='success', response=result, duration_ms=...)
    except Exception as e:
        db.update_run(run_id, status='failed', error=str(e))
        # rely on celery retry if configured
        raise
```

Notes:

- Use `acks_late=True` and task timeouts to avoid task loss on worker failure.
    
- Ensure workers are in a separate autoscaling group from the scheduler.
    

---

## Retry, Backoff, Dead Letter Flow

- Use per-schedule `retry_policy` (max_attempts, backoff_seconds, backoff_type).
    
- When a run fails, update `attempt` in `schedule_runs` and schedule a retry by re-enqueueing the task after backoff (Celery ETA or separate delayed queue).
    
- If attempts exceed `max_attempts`, mark run as `dead_letter` and optionally call an admin webhook / alert.
    

---

## API endpoints (minimal)

- `POST /schedules` — create schedule (payload, cron_expr / interval, retry policy)
    
- `GET /schedules` — list schedules (filter by tenant/user)
    
- `GET /schedules/{id}` — get schedule
    
- `PUT /schedules/{id}` — update schedule
    
- `POST /schedules/{id}/pause` — pause
    
- `POST /schedules/{id}/resume` — resume
    
- `POST /schedules/{id}/run-now` — trigger immediate run
    
- `GET /schedules/{id}/runs` — list run history
    
- `GET /runs/{run_id}` — get run details
    
- `POST /webhook/test` — test webhook execution
    

Include OpenAPI docs and RBAC for tenant isolation.

---

## UI features (React)

- Schedule creation wizard (cron expression helper, human readable preview)
    
- List view with `next_run_at`, `last_run_at`, status, created_by
    
- Run history with filters and JSON response viewer
    
- Per-schedule retry configuration, pause/resume buttons
    
- Webhook response viewer and raw logs download
    
- Quotas & usage dashboard per tenant
    

---

## Observability & Monitoring

- **Metrics**: schedule_count, enqueued_per_minute, success_rate, failure_rate, average_duration, queue_size, worker_count.
    
- **Tracing**: trace scheduler -> enqueue -> worker -> webhook (OpenTelemetry).
    
- **Logs**: structured logs with `schedule_id` and `run_id` fields.
    
- **Alerts**: high failure rates, scheduler not picking jobs, queue backlog.
    

---

## Scalability considerations

- Use batching and pagination when querying due schedules.
    
- Use partitioned tables for `schedule_runs` if run history grows huge.
    
- Shard tenants across scheduler instances if one tenant is noisy.
    
- Use read replicas for dashboard queries.
    
- If per-run latency matters, support "run-now" immediate route that directly enqueues the job.
    

---

## Security

- Sanitize webhook payloads and validate target URLs.
    
- Allow only whitelisted domains / support user-provided SSL certs or OAuth where necessary.
    
- Rate-limit per-tenant runs to avoid abuse.
    
- Encrypt sensitive fields (signing secrets for webhook payloads) at rest.
    

---

## Deployment & HA

- Run scheduler as a StatefulSet (k8s) with leader-election or rely on DB locks.
    
- Run multiple scheduler replicas for failover but ensure locking prevents double-exec.
    
- Workers in autoscaling groups based on Celery queue length.
    
- Run periodic maintenance job to detect stuck runs and requeue.
    

---

## Sample next steps

1. Prototype a minimal PoC: DB schema + single scheduler + one Celery worker + API to create a cron schedule and observe runs.
    
2. Add locking strategy and test failover by killing scheduler instance.
    
3. Build dashboard features and run-history CRUD.
    
4. Add metrics, tracing, and alert rules.
    

---

## Appendix — Helpful libraries & snippets

- `croniter` — compute next cron run
    
- `python-dateutil` — robust timezone handling
    
- `sqlalchemy` — ORM for MySQL
    
- `redis-py` / `aioredis` — for distributed locks or streams
    
- `celery` — execution engine
    
- `alembic` — DB migrations
    
- `opentelemetry-python` — tracing
    

---

_If you want, I can now generate the SQLAlchemy models, the FastAPI endpoints, and the scheduler process code (polling variant) as a single repo scaffold._