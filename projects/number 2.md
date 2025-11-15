# schedular

### You will build:

✔ Create a schedule  
✔ Run a webhook at the scheduled time  
✔ Logs for each execution  
✔ Retries  
✔ Dashboard  
✔ API keys  
✔ Authentication  
✔ Basic UI (Next.js)

No fancy workflow builder, no regions, no code SDK yet.

Let’s break into **phases → sprints → tasks**.

---

# 🧩 **PHASE 1 — Core Backend (FastAPI + Postgres + Celery)**

Time: 1 week  
Goal: Get the cron engine working.

---

## **1. Setup Project Structure**

-  Create FastAPI project skeleton
    
-  Setup Poetry or PDM
    
-  Setup Docker + docker-compose
    
-  Setup `celery` worker container
    
-  Setup `redis` for queue
    
-  Setup Postgres container
    

---

## **2. Database Design**

Create tables:

-  `users`
    
-  `api_keys`
    
-  `schedules`
    
-  `execution_logs`
    
-  `retry_queue` (optional for MVP)
    

---

## **3. Cron Parsing + Scheduler**

-  Add cron parser (`croniter`)
    
-  Implement scheduler service that:
    
    -  runs every 10 seconds
        
    -  finds due tasks
        
    -  enqueues them to Celery
        

---

## **4. Celery Worker Logic**

-  Worker receives a task
    
-  Does HTTP request to target webhook
    
-  Saves execution log
    
-  If failed → retry X times
    
-  If retries exhausted → mark failed
    

---

## **5. Execution Logging**

-  Table: timestamp, status, latency, response code
    
-  Add 7-day retention cron job
    
-  Add API to fetch logs for dashboard
    

---

# 🧩 **PHASE 2 — API Layer**

Time: 4–5 days  
Goal: Make the backend usable for customers.

---

## **6. Auth & API Keys**

-  JWT-based user auth
    
-  API key creation
    
-  API key authentication middleware
    

---

## **7. REST Endpoints**

Build these:

-  POST `/schedules` (create schedule)
    
-  GET `/schedules` (list)
    
-  GET `/schedules/{id}`
    
-  PATCH `/schedules/{id}` (pause/resume/edit)
    
-  DELETE `/schedules/{id}`
    
-  GET `/logs?schedule_id=xxx`
    

---

## **8. Webhook Validation**

-  Test webhook URL on creation
    
-  Prevent localhost/internal URL abuse
    
-  (Optional) Add secret header validation
    

---

# 🧩 **PHASE 3 — Dashboard (Next.js)**

Time: 4–6 days  
Goal: Have a minimum UI like Cronhooks.

---

## **9. Frontend Setup**

-  Create Next.js app
    
-  Setup Auth UI
    
-  Connect API keys
    
-  Install Tailwind
    
-  Base layout + navbar
    

---

## **10. Screens**

-  **Schedules Page**
    
    - List schedules
        
    - Create new schedule
        
    - Edit schedule
        
    - Pause/resume
        
-  **Logs Page**
    
    - Show logs in table
        
    - Status color indicators
        
-  **API Keys Page**
    
    - Generate/delete keys
        

---

## **11. UI Polish**

-  Add toast messages
    
-  2–3 clean cards
    
-  Use shadcn UI components
    

---

# 🧩 **PHASE 4 — DevOps & Infra**

Time: 3–4 days  
Goal: Deploy.

---

## **12. Deploy Infra (Hetzner)**

-  Create 2 VPS (API + Worker)
    
-  Add Traefik or Nginx
    
-  Setup Postgres (managed or self-hosted)
    
-  Add Redis
    
-  Add log retention cron
    

---

## **13. CI/CD**

-  GitHub Actions:
    
    -  Lint
        
    -  Test
        
    -  Push Docker image
        
    -  Deploy via SSH or webhook
        

---

## **14. Domain & HTTPS**

-  Cloudflare DNS
    
-  SSL via Traefik
    
-  Add rate-limiting
    

---

# 🧩 **PHASE 5 — Documentation & Launch**

Time: 2–3 days

---

## **15. Public Docs**

-  Markdown API docs
    
-  Quickstart guide
    
-  cURL examples
    
-  Python example
    
-  Node example
    

---

## **16. Marketing**

-  Create landing page
    
-  Add features + pricing
    
-  Product Hunt draft
    
-  IndieHackers launch post
    
-  Reddit post (devops, webdev)
    

---

# 🧨 **Week-by-Week Timeline (Super realistic)**

### **Week 1: Backend core**

- Scheduler
    
- Worker
    
- Cron parsing
    
- Logs
    

### **Week 2: API + Auth + Testing**

- CRUD
    
- API keys
    
- Webhook execution working
    

### **Week 3: Frontend dashboard**

- UI pages
    
- Create/edit schedule
    
- Logs
    
- Auth
    

### **Week 4: Infra + deploy + docs + launch**

- Hetzner deployment
    
- DNS
    
- Launch content
    

---

# 🌟 **Final MVP Deliverable**

By end of month, you will have a working SaaS where:

- Anyone can create a schedule
    
- Your system hits their webhook
    
- Logs are shown
    
- Retries work
    
- Dashboard works
    
- API keys work
    
- You can charge money
    

**Exactly like early Cronhooks.**