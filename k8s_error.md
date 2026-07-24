# DevBoard: Kubernetes Debugging Retrospective

A record of the issues encountered while deploying the DevBoard app (Go backend + React/Vite frontend + PostgreSQL) to Kubernetes, with interview-ready explanations for each.

---

## Issue 1: Backend/Frontend Port Mismatch

**Symptom:**
```
[vite] http proxy error: /projects
Error: connect ECONNREFUSED 10.96.121.221:8080
```

**Root cause:** The Vite dev server's proxy config was set to forward `/api` requests to port `8080`, but the Go backend was actually listening on `8081` (visible in its startup log: `listening on :8081`). The two configs had drifted out of sync.

**Fix:** Aligned the backend's listen port, the Kubernetes Service's `targetPort`, and the Vite proxy target so all three agreed.

**Interview-ready answer:**
> "I hit a connection-refused error where my frontend couldn't reach the backend API. I traced it by comparing the backend pod's startup logs, which showed it listening on port 8081, against the Vite proxy config, which was pointing at 8080. It was a simple but easy-to-miss configuration drift between two files that both needed to describe the same port. I fixed it by making the port a single source of truth — the backend's actual listen port — and updated the proxy and Service manifest to match, rather than patching just one side."

---

## Issue 2: Stale Docker-Compose Hostname in Kubernetes

**Symptom:**
```
[vite] http proxy error: /projects
Error: getaddrinfo ENOTFOUND backend
```

**Root cause:** The proxy config referenced the hostname `backend`, which was valid under the old `docker-compose` setup (where compose auto-generates short service-name DNS entries) but had never been updated for the Kubernetes migration. In-cluster DNS only resolves actual Kubernetes Service names — in this case `backend-service` — so `backend` failed to resolve entirely.

**Fix:** Updated the proxy target to `http://backend-service:8081`, matching the real Service name and exposed port confirmed via `kubectl get svc`.

**Interview-ready answer:**
> "After fixing the port issue, I hit a DNS resolution error — `ENOTFOUND backend`. I checked `kubectl get svc` and found the actual backend Service was named `backend-service`, not `backend`. It turned out this was legacy config left over from an earlier docker-compose setup, where short service names resolve automatically, but that doesn't carry over to Kubernetes DNS. I updated the proxy target to the real Service name and made a note to avoid hardcoding infrastructure names that can drift when the platform changes underneath the app."

---

## Issue 3: Service Port vs. Container Port Confusion

**Root cause:** Kubernetes Services expose a `port` (what other pods/clients connect to) that is separate from the pod's `targetPort` (the actual container listen port). The Service was correctly configured (`port: 8081` → `targetPort: 8080`), but this distinction wasn't initially obvious when comparing pod logs to Service specs, leading to some confusion about "which port is the real one."

**Interview-ready answer:**
> "I learned to be precise about the difference between a Service's exposed port and the pod's actual container port — they don't have to match, and Kubernetes translates between them via `targetPort`. When debugging connectivity, I made a habit of checking both `kubectl get svc -o yaml` and the pod's own logs, rather than assuming they used the same port number."

---

## Issue 4: Browser Serving a Stale Cached Page

**Symptom:** Visiting `localhost:8080` via `kubectl port-forward` showed a generic nginx welcome page, even after the tunnel and backend were confirmed working.

**Root cause:** Not an infrastructure issue at all — `curl -v http://localhost:8080` returned the correct app HTML with a clean `200 OK`, proving the tunnel and pod were serving correctly. The browser itself was displaying a cached/stale response.

**Fix:** Hard refresh / cleared site data in the browser.

**Interview-ready answer:**
> "One of the trickiest parts of this was a red herring — my browser kept showing an nginx placeholder page even after I'd fixed the actual networking issues. Instead of assuming the infrastructure was still broken, I used `curl -v` to test the endpoint directly, bypassing the browser entirely. That confirmed the server was responding correctly, which told me the problem had moved from infrastructure to the browser's cache. It reinforced a debugging principle I try to follow: isolate each layer independently — don't assume the browser and the server are reporting the same truth."

---

## Issue 5: Foreign Key Violation Due to Empty Seed Data

**Symptom:**
```
[GIN] 500 | POST "/tasks"
ERROR: pq: insert or update on table "tasks" violates foreign key constraint "tasks_project_id_fkey"
```

**Root cause:** The frontend attempted to create a task referencing `project_id: 1`, but the `projects` table was completely empty. The Postgres init ConfigMap only contained DDL (`CREATE TABLE`, indexes, triggers) — no `INSERT` statements — so a fresh database has correct schema but zero rows.

**Fix:** Created a project via `POST /projects` first, satisfying the foreign key before creating a task.

**Interview-ready answer:**
> "The final error was a foreign key violation on task creation. I queried the database directly and found the `projects` table was empty — zero rows, but the table existed, which told me the schema-init step had worked correctly and this wasn't an infra problem. Looking at the init ConfigMap, I realized it only contained DDL — table and index creation — with no seed data. The application logic was working exactly as designed; there just wasn't a valid project for the task to reference yet. I created one via the API to unblock testing, and flagged that a proper seed script (or a migration tool) would be a better long-term solution than relying solely on one-time init scripts."

---

## Bonus Learning: Init Scripts Only Run Once

**Context:** Postgres only executes files in `/docker-entrypoint-initdb.d/` the very first time the data directory is empty (confirmed by the log line `"database system... appears to contain a database; Skipping initialization"`). Editing the ConfigMap later and redeploying does **not** re-run it against an existing PersistentVolumeClaim.

**Interview-ready answer:**
> "This debugging session taught me an important operational detail about Postgres in Kubernetes: init scripts mounted via `/docker-entrypoint-initdb.d` are one-time-only, tied to the *first* initialization of the data directory. Since I was using a PVC for persistence, any future schema changes wouldn't automatically apply on redeploy — I'd need either to wipe the volume (losing data) or introduce a proper migration tool like `golang-migrate` or `goose` for anything beyond the initial bootstrap. It's a good example of why persistence and schema evolution need to be planned together, not treated as an afterthought."

---

## Summary: The Debugging Approach

**Interview-ready answer (if asked to summarize your general approach):**
> "This was a multi-layered issue where each fix revealed the next problem underneath it — port mismatch, then DNS resolution, then a Service/pod port distinction, then a browser caching red herring, then finally a data-layer constraint. My approach throughout was to isolate each layer independently rather than guessing: checking pod logs against Service specs, using `curl` to bypass the browser, querying the database directly instead of trusting the API's error message alone, and confirming each fix with an endpoints/logs check before moving to the next layer. It's a good example of systematic elimination — treating an error message as a starting point for investigation, not a final diagnosis."
