# Therap (BD) Ltd — DevOps Written Exam Prep Sheet

**Exam:** Thursday, July 2, 2026 · 10:00 AM · Banani, Dhaka
**Candidate:** Biprodas Barai
**Prep window:** 4 days

---

## 0. What to expect (exam intel)

Therap's written round is the **initial screening** — usually handwritten or typed in a shared Google Doc, closed-book, ~1.5–2 hours. Based on how they run engineering exams:

- It mixes **conceptual short-answer** + **scenario/troubleshooting** + **a bit of problem-solving/scripting**.
- **This is a production-support / SRE-leaning admin role** (per the actual JD), not a platform/IaC role. Weight your prep accordingly:
  - **CORE (heavy weight):** Linux administration & performance tuning, **JVM/Java app troubleshooting**, production debugging (logs + metrics + root cause), Bash scripting, CI/CD maintenance.
  - **NICE-TO-HAVE (lighter):** Docker/Kubernetes ("home labs count" = no deep prod K8s expected), Prometheus/Grafana.
- ⚠️ **Re-calibration from earlier:** your Kubernetes and Terraform depth is the *nice-to-have* column here. Don't over-invest there. The exam will lean on **Linux internals, JVM debugging, and production incident scenarios.**
- They value **fundamentals over tool-name-dropping**. Explaining *why* beats listing tools.
- The JD literally says "work through problems instead of avoiding them" — frame any incident/scenario answer around **calm, structured, log-first debugging and ownership.**
- They sometimes pull questions from your **CV/projects**, so be ready to explain your production Linux, debugging, and CI/CD work crisply.
- Difficulty is **moderate** (~2.9/5 historically). It's a filter, not a trap.

**Strategy:** Write tight, correct, structured answers. Use a real command or example wherever you can — it signals hands-on experience, which is exactly what separates you from book-only candidates.

---

## 1. Linux & System Administration

This is the backbone. Expect 4–6 questions here.

### Core concepts to nail

**Processes & signals**
- A process has a PID, parent PID, state (Running, Sleeping, Stopped, Zombie).
- `kill -9` (SIGKILL) cannot be caught/ignored; `kill -15` (SIGTERM) is the graceful default.
- **Zombie process:** child finished but parent hasn't read its exit status (`wait()`). Fix: reap via parent, or kill parent so `init`/`systemd` adopts and reaps it.
- **Orphan process:** parent died first; adopted by PID 1.

**Permissions**
- `rwx` = 4/2/1. `chmod 755` = owner rwx, group r-x, others r-x.
- `chmod 644` for files, `755` for directories/executables is the common default.
- **SUID (4000):** run as file owner (e.g. `passwd`). **SGID (2000):** run as group / inherit group on dirs. **Sticky bit (1000):** only owner can delete in a shared dir (e.g. `/tmp`).
- `chown user:group file`, `chmod u+x file`.

**Memory & performance triage** — *very likely a scenario question*
- Order of attack: `top`/`htop` → `free -h` → `df -h` (disk full?) → `iostat` (disk I/O) → `vmstat` → `dmesg` (kernel/OOM messages).
- **Load average** = runnable + uninterruptible processes over 1/5/15 min. Compare against core count: load 4 on a 4-core box = fully utilized, not necessarily bad.
- **OOM killer:** kernel kills a process when memory is exhausted; check `dmesg | grep -i oom`.

**Disk**
- `df -h` (filesystem usage), `du -sh /path` (directory size).
- Classic trap: `df` shows full but `du` doesn't add up → a **deleted file still held open by a process**. Find with `lsof | grep deleted`, restart the holding process.
- Inodes can fill up even with free space: `df -i`.

### Likely questions
1. *"A server's disk shows 100% full but you can't find large files. What's happening?"* → deleted-but-open file held by a process (the `lsof | grep deleted` answer above).
2. *"Difference between hard link and soft link?"* → Hard link points to same inode (same file, survives original deletion, same filesystem only). Soft/symlink points to a path (breaks if target deleted, can cross filesystems).
3. *"How do you find which process is using port 8080?"* → `ss -ltnp | grep 8080` or `lsof -i :8080`.
4. *"Difference between `>` and `>>`?"* → overwrite vs append. And `2>&1` redirects stderr to stdout.

### Commands to have at your fingertips
```
ps aux | grep node          # find a process
ss -ltnp                    # listening TCP ports + PIDs (modern netstat)
journalctl -u nginx -f      # tail systemd service logs
systemctl status/start/stop/enable nginx
grep -r "ERROR" /var/log/   # recursive search
find / -size +100M 2>/dev/null   # find big files
tail -f /var/log/syslog
chmod / chown / chgrp
crontab -e                  # schedule jobs
```

---

## 1.5 JVM / Java Application Troubleshooting ⭐ (HIGH PRIORITY)

**The JD calls this out twice as core, and it's the topic most candidates are weak on — so it's your biggest opportunity to stand out.** You don't need to write Java; you need to understand how a JVM app *behaves and fails* in production.

### JVM memory model
- **Heap:** where objects live. Split into **Young Gen** (Eden + two Survivor spaces) and **Old/Tenured Gen**. New objects start in Young; long-lived ones get promoted to Old.
- **Stack:** per-thread, holds method frames & local variables. A runaway recursion → `StackOverflowError`.
- **Metaspace** (Java 8+, replaced PermGen): class metadata. Lives in native memory, not heap.
- **Key flags:** `-Xms` (initial heap), `-Xmx` (max heap), `-Xss` (thread stack size). A classic gotcha: setting `-Xmx` too high on a small box → OS OOM-kills the JVM.

### Garbage Collection (GC)
- GC reclaims unreachable objects automatically. **"Stop-the-world" (STW)** = app threads pause during certain GC phases.
- **Minor GC** = cleans Young Gen (frequent, fast). **Major/Full GC** = cleans Old Gen (rarer, slower, longer STW).
- **Symptom of trouble:** frequent Full GCs / long STW pauses → app feels frozen, high CPU, latency spikes. Often signals a **memory leak** (Old Gen keeps filling, can't be cleared).
- Collectors to *name*: G1 (default modern), Parallel (throughput), ZGC/Shenandoah (low-latency). You don't need deep tuning — just know they exist and trade throughput vs pause time.

### The errors you MUST be able to diagnose
| Error | Meaning | First move |
|---|---|---|
| `OutOfMemoryError: Java heap space` | heap exhausted (leak or undersized `-Xmx`) | heap dump (`jmap`), check `-Xmx`, look for leak |
| `OutOfMemoryError: Metaspace` | too many classes loaded (classloader leak, often in app servers on redeploy) | check Metaspace size, classloader leaks |
| `OutOfMemoryError: GC overhead limit exceeded` | JVM spending >98% time in GC, reclaiming <2% | almost always a memory leak |
| `StackOverflowError` | stack exhausted — usually infinite recursion | read the stack trace, find the recursive call |
| `OutOfMemoryError: unable to create native thread` | OS thread limit hit (thread leak) | check thread count, `ulimit` |

### Diagnostic toolkit (know what each does)
```
jps                 # list running JVM processes + PIDs
jstat -gcutil <pid> 1000   # live GC stats every 1s (watch Old Gen % climb = leak)
jmap -heap <pid>          # heap summary
jmap -dump:live,format=b,file=heap.hprof <pid>   # heap dump → analyze in Eclipse MAT / VisualVM
jstack <pid>              # thread dump — find deadlocks & stuck threads
jcmd <pid> Thread.print   # modern way to get a thread dump
jcmd <pid> GC.heap_info   # modern heap info
```
- **Heap dump** = snapshot of all objects → find *what's leaking* (analyze offline with Eclipse MAT).
- **Thread dump** = snapshot of all thread states → find *deadlocks* (two threads each holding a lock the other needs → both BLOCKED) and *stuck/hot threads* (high CPU → take several dumps, see which thread stays RUNNABLE in the same spot).

### Production failure patterns (likely scenario questions)
1. *"A Java app is using 100% CPU. How do you investigate?"*
   → `top` to confirm + get PID → `top -H -p <pid>` to find the hot **thread** → convert that thread's TID to hex → take a `jstack`/`jcmd` thread dump → find the matching `nid=0x...` in the dump to see exactly what code is spinning. Common causes: GC thrashing (cross-check `jstat`) or an infinite loop.
2. *"A Java app keeps crashing with OOM. Walk me through it."*
   → Confirm which OOM type from logs → is `-Xmx` reasonable for the box? → capture a heap dump (or enable `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=...` so the *next* crash auto-dumps) → analyze dump for the dominant object retainer → fix leak or resize heap.
3. *"App is slow / freezing intermittently."*
   → Check GC logs (`-Xlog:gc` / `-verbose:gc`) for long Full GC pauses → if Old Gen never drops after Full GC, it's a leak. → Also rule out thread contention via thread dumps.
4. *"App won't start."* → read the stack trace top-to-bottom; `Caused by:` at the bottom is usually the real root cause (port in use, missing config/env var, DB unreachable, classpath issue).

### Where to look (Java app servers)
- App logs: often `catalina.out` (Tomcat), or `logs/` under the app server; Spring Boot apps log to stdout/configured file.
- Enable **GC logging** proactively: `-Xlog:gc*:file=gc.log` (Java 9+) or `-XX:+PrintGCDetails` (older).
- `-XX:+HeapDumpOnOutOfMemoryError` should be standard in prod — mention this; it shows ops maturity.

### One-liner that impresses
"For a JVM incident I'd start with logs to classify the failure, use `jstat` to see if it's GC-driven, take a thread dump for CPU/deadlock issues or a heap dump for memory issues, and analyze offline — rather than just restarting and hoping." *Say something like this for any JVM scenario.*

---

## 2. Bash Scripting

Expect 1–2 questions, possibly "write a small script."

### Must-know syntax
```bash
#!/bin/bash
set -euo pipefail          # fail fast: exit on error, unset var, pipe failure

# Variables
name="bipro"
echo "Hello $name"

# Conditionals
if [ -f "$file" ]; then echo "exists"; fi
if [ "$a" -gt "$b" ]; then ...; fi   # -eq -ne -lt -gt -le -ge for numbers
if [ "$x" == "yes" ]; then ...; fi   # == for strings

# Loops
for f in *.log; do echo "$f"; done
while read line; do echo "$line"; done < file.txt

# Functions
backup() { cp "$1" "$1.bak"; }

# Exit codes: 0 = success, non-zero = failure. Check with $?
```

### Common interview tasks
- **Count lines in a file:** `wc -l file`
- **Find top 10 IPs in an access log:** `awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10`
- **Disk-usage alert script:**
```bash
#!/bin/bash
THRESHOLD=80
usage=$(df / | awk 'NR==2 {print $5}' | tr -d '%')
if [ "$usage" -gt "$THRESHOLD" ]; then
  echo "Disk usage critical: ${usage}%"
fi
```

### Key idioms they like to test
- `$0` = script name, `$1`–`$9` = args, `$@` = all args, `$#` = arg count, `$?` = last exit code, `$$` = current PID.
- `&&` runs next only on success; `||` runs next only on failure.
- Difference between `$(cmd)` (preferred) and backticks `` `cmd` `` (legacy) for command substitution.

---

## 3. Networking

DevOps networking questions are common and you should be strong here.

### Concepts
- **OSI vs TCP/IP:** be able to name the 7 OSI layers (Physical, Data Link, Network, Transport, Session, Presentation, Application) and map common protocols.
- **TCP vs UDP:** TCP = connection-oriented, reliable, ordered, 3-way handshake (SYN, SYN-ACK, ACK). UDP = connectionless, fast, no guarantee (DNS, video, VoIP).
- **3-way handshake** and **4-way termination** (FIN/ACK) — be ready to draw.
- **Common ports:** 22 SSH, 80 HTTP, 443 HTTPS, 53 DNS, 25/587 SMTP, 3306 MySQL, 5432 Postgres, 6379 Redis, 27017 Mongo.
- **DNS resolution flow:** browser cache → OS cache → resolver → root → TLD → authoritative. Record types: A, AAAA, CNAME, MX, TXT, NS.
- **HTTP status codes:** 2xx success, 3xx redirect, 4xx client error (401 unauthorized vs 403 forbidden vs 404 not found), 5xx server error (502 bad gateway, 503 unavailable, 504 timeout).
- **Subnetting basics:** CIDR `/24` = 256 addresses (254 usable). Private ranges: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`.

### Likely scenario
*"A user can't reach your web app. Walk me through debugging."*
→ `ping` (host up / DNS resolving?) → `nslookup`/`dig` (DNS correct?) → `curl -v` (does the server respond? what status?) → `ss -ltnp` (is the service listening?) → check firewall/security group → check reverse proxy (nginx) logs → check app logs. Narrate it layer by layer; that structure is what they're grading.

### Commands
```
ping host / traceroute host
dig example.com / nslookup example.com
curl -v https://api.example.com
telnet host 443 / nc -zv host 443   # is the port open?
ss -tunap                            # all connections
ip a / ip route                      # interfaces & routing
```

---

## 4. Docker & Containers

High-probability section. Know the difference between concepts deeply.

### Core
- **Container vs VM:** containers share the host kernel (lightweight, fast, MB-sized), VMs run a full guest OS via hypervisor (heavier, GB-sized, stronger isolation).
- **Image vs Container:** image = read-only template (built from layers); container = running instance of an image.
- **Layers & caching:** each Dockerfile instruction = a layer. Order matters — put rarely-changing steps (install deps) before frequently-changing ones (copy source) to maximize cache reuse.

### Dockerfile literacy
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./        # copy deps first for layer caching
RUN npm ci --production
COPY . .                     # source changes don't bust the deps layer
EXPOSE 3000
CMD ["node", "server.js"]
```
- **`CMD` vs `ENTRYPOINT`:** ENTRYPOINT = fixed executable; CMD = default args (overridable at run). Common pattern: ENTRYPOINT sets the binary, CMD sets default flags.
- **`COPY` vs `ADD`:** prefer COPY; ADD also untars and fetches URLs (usually unwanted).
- **`RUN` vs `CMD`:** RUN executes at build time (creates a layer); CMD at container start.

### Must-know answers
- **Multi-stage builds:** build in a heavy image, copy only the artifact into a slim final image → smaller, more secure images. Be ready to explain — it's a favorite.
- **Reduce image size:** alpine base, multi-stage, `.dockerignore`, combine RUN layers, remove caches.
- **Volumes vs bind mounts:** volumes are Docker-managed (`/var/lib/docker/volumes`), bind mounts map a host path. Volumes for persistent data (DBs), bind mounts for dev.
- **Container networking:** bridge (default), host, none, overlay (multi-host/swarm). Containers on the same bridge network resolve each other by name.

### Commands
```
docker build -t app:1.0 .
docker run -d -p 8080:80 --name web app:1.0
docker ps -a / docker logs -f web / docker exec -it web sh
docker images / docker rmi / docker system prune
docker-compose up -d
```

---

## 5. Kubernetes

This is your strength — make it shine. Likely 2–4 questions for a platform-leaning DevOps role.

### Architecture (be ready to sketch)
- **Control plane:** API server (front door), etcd (cluster state KV store), scheduler (assigns pods to nodes), controller manager (reconciliation loops).
- **Worker node:** kubelet (runs pods, talks to API server), kube-proxy (networking/iptables rules), container runtime (containerd).

### Core objects
- **Pod:** smallest deployable unit; one or more containers sharing network/storage.
- **ReplicaSet:** maintains N pod replicas.
- **Deployment:** manages ReplicaSets, enables rolling updates & rollback. *(What you actually deploy with.)*
- **Service:** stable networking/load-balancing to pods. Types: **ClusterIP** (internal), **NodePort** (port on every node), **LoadBalancer** (cloud LB), **ExternalName**.
- **Ingress:** L7 HTTP routing (host/path-based) to services, via an ingress controller (nginx, traefik).
- **ConfigMap / Secret:** externalize config / sensitive data.
- **StatefulSet:** for stateful apps needing stable identity & storage (DBs).
- **DaemonSet:** one pod per node (log/metric agents).
- **Namespace:** logical isolation.

### High-probability questions
1. *"How does a rolling update work and how do you roll back?"* → Deployment gradually replaces old pods with new (`maxSurge`/`maxUnavailable` control pace); `kubectl rollout undo deployment/x` to revert.
2. *"Liveness vs Readiness probe?"* → **Liveness** = is it alive? fail → restart container. **Readiness** = is it ready for traffic? fail → remove from Service endpoints (no restart). This distinction is a classic.
3. *"A pod is stuck in `CrashLoopBackOff` — debug it."* → `kubectl describe pod x` (events, last state, exit code), `kubectl logs x --previous`, check probes/resource limits/config/secrets. Narrate the flow.
4. *"Requests vs Limits?"* → requests = guaranteed/scheduled amount; limits = hard cap (CPU throttled, memory over-limit → OOMKilled).
5. *"How do pods discover each other?"* → via Services + cluster DNS (`service.namespace.svc.cluster.local`).

### Commands
```
kubectl get pods -o wide / kubectl get svc,deploy,ingress
kubectl describe pod <name>
kubectl logs -f <pod> [-c container] [--previous]
kubectl exec -it <pod> -- sh
kubectl apply -f manifest.yaml
kubectl rollout status/undo deployment/<name>
kubectl scale deployment/<name> --replicas=5
kubectl get events --sort-by=.lastTimestamp
```

---

## 6. CI/CD

Definitional + design questions. You have real pipeline experience — lean on it.

### Concepts
- **CI (Continuous Integration):** devs merge frequently; every push triggers automated build + tests. Goal: catch integration issues early.
- **CD (Delivery vs Deployment):** Delivery = every change is *ready* to deploy (manual approval gate). Deployment = every passing change auto-ships to prod.
- **Pipeline stages:** source → build → test (unit/integration) → security scan → artifact/package → deploy (staging → prod) → monitor.
- **Artifacts:** build outputs (Docker images, jars) stored in a registry (ECR, Docker Hub, Nexus).

### Deployment strategies — *know all four cold*
| Strategy | How | Trade-off |
|---|---|---|
| **Rolling** | replace instances gradually | no downtime, slow, mixed versions briefly |
| **Blue-Green** | two identical envs, switch traffic | instant rollback, doubles infra cost |
| **Canary** | route small % to new version, ramp up | safest, needs good monitoring/routing |
| **Recreate** | kill all old, start new | downtime, simplest |

### Likely questions
- *"How do you keep secrets out of a pipeline?"* → secret managers (Vault, AWS Secrets Manager, GitHub/GitLab secrets), never hard-code, never commit. **Tie this to your real fix** where a live API key was hard-coded in a repo and needed revocation + git history purge — that's a great war story to mention.
- *"What makes a good pipeline?"* → fast feedback, fail-fast, idempotent, reproducible builds, automated tests/gates, easy rollback.
- *"Difference between Jenkins, GitHub Actions, GitLab CI?"* → Jenkins (self-hosted, plugin-heavy, flexible), GitHub Actions (YAML, tightly integrated with GitHub, runner-based), GitLab CI (built-in, `.gitlab-ci.yml`).

---

## 7. Infrastructure as Code (Terraform / Ansible)

You know this — be precise on the distinctions.

- **IaC benefits:** versioned, repeatable, reviewable, auditable infra; eliminates config drift and snowflake servers.
- **Terraform (provisioning) vs Ansible (configuration):** Terraform *creates* infra (declarative, state-based, immutable-leaning). Ansible *configures* existing servers (procedural-ish, agentless over SSH, YAML playbooks). They're complementary.
- **Declarative vs Imperative:** declarative = describe desired end state (Terraform, K8s); imperative = describe steps (a bash script).
- **Terraform state:** `terraform.tfstate` maps config to real resources. Store **remotely** (S3 + DynamoDB lock) for teams — never commit it (contains secrets, causes conflicts).
- **Core commands:** `terraform init` → `plan` (preview diff) → `apply` → `destroy`.
- **Idempotency:** running the same config repeatedly converges to the same state without side effects — central to both tools.

### Likely question
*"You changed infra manually in the console. What happens on next `terraform apply`?"* → drift; Terraform detects the diff and tries to revert to match the config. Lesson: don't make manual changes to IaC-managed resources.

---

## 8. Monitoring & Observability

Their stack lists **Prometheus + Grafana** — expect at least one question.

- **Three pillars of observability:** **Metrics** (numeric time-series), **Logs** (discrete events), **Traces** (request path across services).
- **Prometheus:** pull-based metrics, scrapes `/metrics` endpoints, stores time-series, queried with **PromQL**. Alertmanager handles alerts.
- **Grafana:** visualization/dashboards on top of Prometheus (and others).
- **Metric types:** Counter (only goes up — requests), Gauge (up/down — memory), Histogram/Summary (distributions — latency percentiles).
- **Monitoring vs Observability:** monitoring = watching known failure modes (dashboards/alerts); observability = ability to ask *new* questions about unknown issues from your telemetry.
- **The four golden signals** (Google SRE): **Latency, Traffic, Errors, Saturation.** Memorize these — it's a frequent question.
- **SLI / SLO / SLA:** Indicator (measured metric) → Objective (internal target, e.g. 99.9%) → Agreement (external contract with penalties).

### Likely question
*"What would you monitor for a web service?"* → the four golden signals + host-level (CPU/mem/disk) + business metrics. Tie back to error budgets if you want to impress.

---

## 9. Git

Quick but near-guaranteed.

- **merge vs rebase:** merge preserves history with a merge commit; rebase rewrites commits onto a new base for linear history. **Never rebase shared/public branches.**
- **`git reset` vs `revert`:** reset moves the branch pointer (rewrites history — `--soft` keeps changes staged, `--hard` discards); revert creates a *new* commit undoing a change (safe for shared history).
- **`git fetch` vs `pull`:** fetch downloads only; pull = fetch + merge.
- **Resolving merge conflicts:** `git status` to see conflicts → edit the `<<<<<<<`/`=======`/`>>>>>>>` markers → `git add` → commit.
- **`.gitignore`:** keep secrets, `node_modules`, build artifacts, and state files out of the repo.
- **stash:** `git stash` / `git stash pop` to shelve work-in-progress.

---

## 10. Cloud / AWS (likely lighter, but be ready)

- **Core services:** EC2 (compute), S3 (object storage), RDS (managed DB), VPC (networking), IAM (access), ELB (load balancing), Route 53 (DNS), CloudWatch (monitoring), ECR/EKS/ECS (containers), Lambda (serverless).
- **IAM:** users, groups, roles, policies. **Principle of least privilege.** Prefer roles over long-lived keys.
- **Security group vs NACL:** SG = stateful, instance-level, allow-only. NACL = stateless, subnet-level, allow + deny.
- **Public vs private subnet:** public has a route to an internet gateway; private uses a NAT gateway for outbound only.
- **High availability:** multi-AZ deployments, auto-scaling groups, load balancers.
- **Scaling:** horizontal (more instances) vs vertical (bigger instance). Horizontal is the cloud-native default.

---

## 11. Problem-Solving / DSA (Therap historically includes 1–2)

Even for DevOps, Therap leans on fundamentals. Expect **1 easy/medium coding or logic problem** — you can usually answer in Python or pseudocode.

### Refresh these patterns
- **Arrays/strings:** two-pointers, sliding window, hash maps for counting.
- **Classic problems they've used:** *max profit from one buy/sell* (track min so far, max profit so far — O(n)), *reverse a string with recursion*, *find duplicates* (use a set), *FizzBuzz*, *anagram check* (sort or count chars).
- **Complexity:** be able to state Big-O of your solution. Know O(1) < O(log n) < O(n) < O(n log n) < O(n²).

### Quick template (max profit example)
```python
def max_profit(prices):
    min_price = float('inf')
    best = 0
    for p in prices:
        min_price = min(min_price, p)
        best = max(best, p - min_price)
    return best
```
Practice writing 3–4 of these by hand (no IDE) so the syntax is automatic under exam pressure.

---

## 12. SQL / DBMS basics (often 1 question)

- **JOINs:** INNER (matching rows only), LEFT (all left + matching right), RIGHT, FULL OUTER.
- **Aggregates + GROUP BY + HAVING** (HAVING filters groups; WHERE filters rows before grouping).
- **Index:** speeds up reads, slows writes; know that a query on an unindexed column = full table scan.
- **ACID:** Atomicity, Consistency, Isolation, Durability.
- **Normalization vs denormalization:** reduce redundancy vs optimize read performance.
- **Sample query they might ask** — *"second highest salary":*
```sql
SELECT MAX(salary) FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);
```

---

## 13. System Design / Scenario (the differentiator)

There's often one open-ended design or troubleshooting question. This is where senior candidates separate from juniors. **Always answer with structure, not a wall of tools.**

### Framework for any "design X" question
1. Clarify requirements (scale, traffic, SLAs).
2. High-level components (LB → app servers → cache → DB).
3. Scaling (horizontal, stateless services, DB replicas/sharding).
4. Reliability (redundancy, health checks, auto-recovery).
5. Observability (the four golden signals).
6. Trade-offs (call them out — shows maturity).

### Framework for any "X is broken" question
1. Define the symptom precisely (latency? errors? down?).
2. Check monitoring/dashboards first.
3. Narrow the layer: DNS → LB → app → DB → infra.
4. Read logs at the suspect layer.
5. Form a hypothesis, test it, mitigate (rollback/scale), then root-cause.

Saying *"first I'd look at the dashboards and recent deploys, then narrow by layer"* instantly reads as someone who's done this in production. That's you.

---

## 14. Four-Day Study Plan

| Day | Focus | Goal |
|---|---|---|
| **Sun (today)** | Linux + Bash + production debugging (sections 1, 2, 13) | The heaviest-weighted core. Write commands by hand. Practice the "X is broken" framework out loud. |
| **Mon** | **JVM/Java troubleshooting (1.5)** + Networking (3) | JVM is the make-or-break differentiator and your biggest gap — give it a full focused block. Memorize the OOM table and the 100%-CPU debugging flow. |
| **Tue** | CI/CD + Monitoring + Git (6, 8, 9) + light Docker (4) | Conceptual; tie each to a real project example. Docker stays light, K8s/IaC are nice-to-have — skim, don't grind. |
| **Wed** | DSA + SQL (11–12) + full mock + JVM re-review | Hand-write 3–4 coding problems. Do one timed mock from this sheet. Re-read section 1.5. Sleep early. |

**Priority order if you're short on time:** Linux (1) → JVM (1.5) → production debugging (13) → Bash (2) → CI/CD (6) → everything else. Kubernetes/Terraform are the *lowest* priority for this specific role despite being your strength.

**Each day:** 60% reviewing concepts, 40% writing answers by hand (the exam is written — practice the medium, not just the material).

---

## 15. Day-Of Tips

- Arrive ~30 min early — Banani traffic. Carry printed CV copies, ID, and a couple of pens.
- **Read every question fully before writing.** Allocate time per question; don't sink 40 min into one.
- **Structure each answer:** definition → how it works → a concrete command/example. One real command is worth a paragraph of theory.
- If you don't know something, **show your reasoning** — partial structured thinking earns partial credit.
- For scenarios, **narrate the debugging path layer by layer** rather than jumping to an answer.
- Leave 5 min to review. Legible handwriting counts more than people think.
- You have real production experience most candidates lack — let it show in the examples you pick.

---

## 16. Self-Test (try these closed-book before Wednesday)

1. Disk shows 100% full, `du` doesn't account for it. Why?
2. Liveness vs readiness probe — and what each does on failure.
3. Write a one-liner: top 5 IPs by request count from an access log.
4. Blue-green vs canary — when would you pick each?
5. Pod stuck in `CrashLoopBackOff` — your debugging steps.
6. TCP 3-way handshake — name the packets.
7. `terraform apply` after a manual console change — what happens?
8. The four golden signals.
9. Multi-stage Docker build — what problem does it solve?
10. `git reset` vs `git revert` on a shared branch.
11. A Java app is pinned at 100% CPU — your step-by-step investigation.
12. Name three `OutOfMemoryError` types and what each means.
13. Heap dump vs thread dump — when do you take each?
14. What does a Full GC that doesn't free Old Gen memory tell you?

*(All answers are above — grade yourself.)*

---

**Tui already strong — eta mostly polish ar structure. Confidence ta key. Best of luck, Bipro! 💪**
