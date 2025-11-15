# ✅ **DATABASE SCHEMA (PostgreSQL / MySQL)**

Designed with:

- Multi-tenant (user → projects → jobs → executions)
    
- Resilient distributed scheduling
    
- Lock-free coordination
    
- Execution logging
    
- Cron parsing + next_run caching
    
- Soft future upgrade to retries, backoff, job chaining
    

---

# 🍀 **1. users (from Supabase)**

You won’t maintain this table. Supabase manages:

```sql
auth.users   
id (uuid) PK   
email   
created_at
```

You will store foreign references using `user_id`.

---

# 🍀 **2. projects**

```sql
projects (   
	id            CHAR(36) PRIMARY KEY,   
	user_id       CHAR(36) NOT NULL,  -- supabase user id   
	name          VARCHAR(255) NOT NULL,   
	created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP 
)  
INDEX (user_id)
```

---

# 🍀 **3. jobs**

### Represents each scheduled task.

```sql
jobs (   
	id               CHAR(36) PRIMARY KEY,   
	project_id       CHAR(36) NOT NULL,   
	name             VARCHAR(255) NOT NULL,   
	schedule         VARCHAR(50) NOT NULL,          -- cron string   
	webhook_url      TEXT NOT NULL,   
	timezone         VARCHAR(50) DEFAULT 'UTC',      
	enabled          BOOLEAN DEFAULT TRUE,      
	last_run_at      TIMESTAMP NULL,   
	next_run_at      TIMESTAMP NOT NULL,            -- precomputed cron    
	created_at       TIMESTAMP DEFAULT CURRENT_TIMESTAMP,   
	updated_at       TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP 
) 
INDEX (project_id) 
INDEX (next_run_at)       -- critical for scheduler
INDEX (enabled)
```

👉 **Workers will poll jobs WHERE next_run_at <= now() AND enabled = 1**  
This is fastest and simplest for MVP.

---

# 🍀 **4. job_executions**

### Logs each attempt.

```sql
job_executions (   
	id               CHAR(36) PRIMARY KEY,   
	job_id           CHAR(36) NOT NULL,   
	status           ENUM('pending', 'success', 'failure') NOT NULL,   
	started_at       TIMESTAMP NULL,   
	finished_at      TIMESTAMP NULL,  
	response_code    INT NULL,   
	response_body    MEDIUMTEXT NULL,   
	created_at       TIMESTAMP DEFAULT CURRENT_TIMESTAMP 
)  
INDEX (job_id) INDEX (created_at)
```
---

# 🍀 **5. api_keys (optional MVP)**

Useful if users want server-to-server calls.

`api_keys (   id          CHAR(36) PRIMARY KEY,   project_id  CHAR(36) NOT NULL,   key_hash    CHAR(64) NOT NULL,  -- hashed API key   created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP )  INDEX (project_id)`

---

# 🍀 **6. scheduler_locks (optional but recommended)**

If you build **active-active scheduling** (multiple schedulers):

`scheduler_locks (   key        VARCHAR(64) PRIMARY KEY,   -- "scheduler-global" or "job-{id}"   owner_id   VARCHAR(64),               -- instance id   expires_at TIMESTAMP )`

---

# ✅ REDIS DESIGN (Distributed Scheduling + Work Queue)

Use **Redis** for coordination between scheduler → workers.

The Redis components:

---

# 🍎 **1. Sorted Set: `cron:due_jobs`**

Scheduler pushes due jobs into ZSET:

`key: cron:due_jobs value: job_id score: timestamp (epoch ms)`

Example:

`ZADD cron:due_jobs 1731652500000 "job-123"`

Workers will poll:

`ZRANGEBYSCORE cron:due_jobs -inf now LIMIT 1`

Then pop:

`ZPOPMIN cron:due_jobs`

---

# 🍎 **2. Queue: `queue:jobs` (Redis List / Stream)**

After popping from ZSET, you push for execution:

`LPUSH queue:jobs {job_execution_payload}`

or **better**:

### Prefer Redis Stream (`XADD`) for reliability:

`XADD queue:jobs * job_id=... execution_id=...`

Workers will use:

`XREADGROUP GROUP workers worker-1 STREAMS queue:jobs >`

This gives:

- Acknowledgements
    
- Retry on failure
    
- Visibility timeout
    
- Consumer groups  
    (perfect for distributed workers)
    

---

# 🍎 **3. Distributed Lock For Job Execution**

To prevent race conditions:

`SETNX lock:job:{jobId} worker_id EX 30`

Release lock after execution:

`DEL lock:job:{jobId}`

---

# 🍎 **4. Scheduler Leadership Election**

If you run multiple scheduler nodes:

`SET scheduler:leader instanceId NX EX 5`

Node renews every 3 seconds.  
If leader dies, a new one takes over.

---

# 🍎 **5. Execution Status Updates**

Workers push back execution result through Redis pub/sub:

`PUBLISH execution:result {execution_id, status}`

FastAPI listens and updates the DB.

---

# 📌 Execution Flow (End-to-End)

### **1. Scheduler**

- Fetch jobs from DB where `next_run_at <= now() AND enabled = 1`
    
- Push job → Redis sorted set `cron:due_jobs`
    
- Update `next_run_at` = cron.next()
    

### **2. Worker**

- Pop earliest due job from ZSET
    
- Deduplicate using Redis Lock
    
- Create execution entry in DB (`pending`)
    
- Publish to Redis Stream: `queue:jobs`
    
- Worker consumes → executes webhook
    
- Updates DB with success/failure
    

### **3. Dashboard**

- Polls `/jobs/:id/executions`
    
- Realtime later using Supabase Realtime/WebSockets
    

---

# 💎 Scalability

This design can scale to:

- Millions of scheduled jobs
    
- Hundreds of worker nodes
    
- Multi-region later
    

Cronhooks, Trigger.dev, Temporal, Cronitor—all use variations of this model.