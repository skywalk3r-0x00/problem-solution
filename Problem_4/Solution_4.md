# Production Issue Analysis & Fixes

---

## 1. Problems Identified

| No | File | Issue |
|----|------|------|
| 1 | nginx/conf.d/default.conf | Proxy points to port `3001` while API listens on `3000` → 502 on every request |
| 2 | docker-compose.yml | No healthchecks → API starts before Postgres/Redis are ready → crash loop |
| 3 | docker-compose.yml | `init.sql` not mounted → `max_connections = 20` never applied |
| 4 | docker-compose.yml | No `restart` policy → a single crash kills the service permanently |
| 5 | api/src/index.js | No DB pool size cap + unhandled Redis `error` event crashes Node process |
| 6 | nginx/nginx.conf | Empty config not mounted → `conf.d/default.conf` never loaded |
| 7 | nginx/conf.d/default.conf | `/status` route not proxied → healthcheck endpoint inaccessible |

---

## 2. How Issues Were Diagnosed

| No | Suspicious Components | Diagnosis |
|----|----------------------|----------|
| 1 | nginx `default.conf` | Found port mismatch: `3001` vs `3000` (from API code) |
| 2 | docker-compose.yml | `depends_on` without `condition: service_healthy` → no readiness guarantee |
| 3 | postgres/init.sql | File exists but not mounted in container |
| 4 | Node.js (ioredis) | Unhandled `error` event crashes process (per docs) |
| 5 | NGINX routing | `http://localhost:8080/status` → 404 → traced to missing nginx.conf mount |

---

## 3. Fixes Applied

| No | Fix |
|----|-----|
| 1 | Updated NGINX upstream port `3001 → 3000` |
| 2 | Created `nginx.conf` and mounted it |
| 3 | Added `/status` route in `default.conf` |
| 4 | Added `healthcheck` + `condition: service_healthy` for all services |
| 5 | Replaced `wget` with Node HTTP client for API healthcheck (`node:20-alpine`) |
| 6 | Added `start_period: 15s` to avoid false unhealthy |
| 7 | Mounted `init.sql` into `/docker-entrypoint-initdb.d/` |
| 8 | Added `restart: unless-stopped` to all services |
| 9 | Limited DB pool via `DB_POOL_MAX=5` |
| 10 | Added `redis.on("error", ...)` handler |
| 11 | Added named volume for Postgres persistence |

---

## 4. Monitoring & Alerts

| No | Tools / Alerts |
|----|--------------|
| 1 | Prometheus + Grafana: HTTP 5xx rate, p95 latency, DB pool wait time |
| 2 | Alerts: |
|    | - >1% 5xx error rate |
|    | - Postgres connections > 80% of `max_connections` |
|    | - Redis memory > 80% |
| 3 | Centralized logging (CloudWatch Logs) |
| 4 | Uptime check on `/status` endpoint every 30s |

---

## 5. Prevention Strategy

| No | Points |
|----|-------|
| 1 | Validate compose file in CI (`docker compose config`) |
| 2 | Use staging environment before production |
| 3 | Store secrets in AWS Secrets Manager / Vault (not hardcoded) |
| 4 | Move to Kubernetes with readiness probes |
| 5 | Monitor DB pool + apply auto-scaling rules |

---

## 6. Key Takeaways

- Most failures came from **missing operational safeguards**, not code logic  
- Healthchecks are not optional — they are **contracts between services**  
- Observability should be designed **before incidents happen**  
- Small misconfigurations (ports, mounts) can cascade into full system failure  

---
