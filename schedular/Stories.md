# **User Stories with “What We Want to Build” (MVP Only)**

(You can paste this directly into Jira / Linear.)

---

# **EPIC 1 — Job Management**

---

## **US-1: Create a Scheduled Job**

**User Story:**  
As a developer, I want to create a scheduled job using a cron expression so that it runs automatically at defined times.

**What We Want to Build:**

- API endpoint (`POST /v1/jobs`) that stores job metadata
    
- Cron expression validator
    
- Calculate next run timestamp and store it
    
- Store payload, callback URL, and timezone
    
- Generate a unique job ID
    

**Acceptance Criteria:**

- Invalid cron → return 400
    
- Job saved with next_run time
    
- Returns job ID + job metadata
    
- Enforces “max jobs per workspace” rule
    

---

## **US-2: Create a One-Time Job**

**User Story:**  
As a developer, I want to schedule a job at a specific timestamp so that it executes once.

**What We Want to Build:**

- Support for “run_at” (ISO timestamp)
    
- Store as one-time job
    
- Mark job as inactive after run
    

**Acceptance Criteria:**

- Timestamp validation
    
- Stored with next_run = run_at
    
- After successful execution, job becomes inactive
    

---

## **US-3: Get Job Details**

**User Story:**  
As a developer, I want to fetch job details so that I can confirm the job configuration.

**What We Want to Build:**

- API endpoint (`GET /v1/jobs/{id}`)
    
- Fetch full metadata from DB
    

**Acceptance Criteria:**

- Returns name, schedule, timezone, callback URL, payload, headers
    
- Returns next run + last run status
    

---

## **US-4: Delete a Job**

**User Story:**  
As a developer, I want to delete a scheduled job so that it stops executing.

**What We Want to Build:**

- Soft delete mechanism
    
- API endpoint (`DELETE /v1/jobs/{id}`)
    

**Acceptance Criteria:**

- Deleted job never triggers again
    
- 404 if job not found
    

---

# **EPIC 2 — Distributed Scheduler**

---

## **US-5: Leader Election**

**User Story:**  
As the system, I need to ensure only one scheduler node is active so that jobs are not duplicated.

**What We Want to Build:**

- Redis-based leader election
    
- Heartbeat renewal mechanism
    
- Failover handling when leader dies
    

**Acceptance Criteria:**

- Only one node holds the “leader lock”
    
- Failover within 2–3 seconds
    
- Non-leader nodes become passive
    

---

## **US-6: Cron Parsing & Next Run Calculation**

**User Story:**  
As the scheduler, I need to parse cron expressions so that I can determine the next execution time.

**What We Want to Build:**

- Cron parser integration
    
- Function to compute “next run” timestamp
    
- Support timezones
    

**Acceptance Criteria:**

- Correct next run for cron expressions
    
- Handles timezone properly
    
- Recomputes next run after each execution
    

---

## **US-7: Detect Due Jobs & Enqueue**

**User Story:**  
As the scheduler, I want to scan for due jobs and enqueue them so that workers execute them.

**What We Want to Build:**

- Database query for `next_run <= now()`
    
- Push job ID + metadata to Redis queue
    
- Update job.next_run to next timestamp
    

**Acceptance Criteria:**

- All due jobs added to queue exactly once
    
- next_run updated immediately
    
- Missed schedules (during downtime) are caught up  
    (only _one run_ is queued for MVP)
    

---

# **EPIC 3 — Worker Service**

---

## **US-8: Consume Tasks From Queue**

**User Story:**  
As a worker, I want to pull jobs from the queue so that I can execute webhook tasks.

**What We Want to Build:**

- Redis consumer
    
- Concurrency handling (threads or async)
    
- Internal task execution pipeline
    

**Acceptance Criteria:**

- Worker pulls tasks continuously
    
- Worker logs each execution attempt
    

---

## **US-9: Execute Webhook**

**User Story:**  
As a worker, I want to trigger a callback URL so that the user’s business logic runs.

**What We Want to Build:**

- HTTP POST client
    
- Support custom headers & JSON payload
    
- Timeout of 10 seconds
    
- Capture status code + body
    

**Acceptance Criteria:**

- POST request sent to callback URL
    
- On timeout, mark as failed
    
- Log status, duration, and error
    

---

## **US-10: Retry Logic**

**User Story:**  
As a developer, I want jobs to retry when a webhook fails so that the system is reliable.

**What We Want to Build:**

- Default retry = 3 attempts
    
- Exponential backoff (e.g., 30s → 2m → 10m)
    
- Mark final status if all retries fail
    

**Acceptance Criteria:**

- Retries correctly scheduled
    
- No infinite retry loops
    
- Final failure is logged
    

---

# **EPIC 4 — Logging**

---

## **US-11: Store Logs**

**User Story:**  
As a developer, I want logs for each execution so that I can troubleshoot failures.

**What We Want to Build:**

- DB table for job_logs
    
- Save attempt number, status, response, duration
    

**Acceptance Criteria:**

- Logs created for every run & retry
    
- Logs retrievable via API
    

---

## **US-12: Get Logs**

**User Story:**  
As a developer, I want to fetch logs for a specific job so that I can inspect behavior.

**What We Want to Build:**

- API endpoint (`GET /v1/jobs/{id}/logs`)
    
- Pagination support (MVP: offset-limit)
    

**Acceptance Criteria:**

- Returns logs sorted by timestamp
    
- Includes status + error fields
    

---

# **EPIC 5 — Authentication & Quotas**

---

## **US-13: API Key Authentication**

**User Story:**  
As a developer, I want my requests to be authenticated so that my jobs remain secure.

**What We Want to Build:**

- API key model in DB
    
- `x-api-key` header authentication
    
- Middleware to enforce key
    

**Acceptance Criteria:**

- Missing/invalid key returns 401
    
- Rate-limited brute-force attempts
    

---

## **US-14: Workspace Quota**

**User Story:**  
As a system, I want to limit how many jobs users create so that workloads remain predictable.

**What We Want to Build:**

- Configurable limit: 100 jobs/workspace
    
- Validation in “create job” API
    

**Acceptance Criteria:**

- > 100 jobs returns 429 error
    
- Limit configurable via env var
    

---

# **EPIC 6 — Observability / DevOps**

---

## **US-15: Health Check Endpoint**

**User Story:**  
As an engineer, I want a health check so that I can monitor service uptime.

**What We Want to Build:**

- `/health` endpoint
    
- Checks DB + Redis connection
    

**Acceptance Criteria:**

- `/health` returns 200 when healthy
    
- Returns 500 on failures
    

---

## **US-16: Basic Metrics**

**User Story:**  
As an engineer, I want metrics so that I can track system behavior.

**What We Want to Build:**

- Counters for jobs scheduled, executed, failed
    
- Prometheus-friendly endpoint
    

**Acceptance Criteria:**

- `/metrics` exposes counters
    
- Tested with simple Prometheus setup