---
longform:
  format: single
  title: DevOps General
---
# DevOps Engineer — Interview Prep

### Quick-reference guide for all 3 technical rounds

---

## ROUND 1: Fundamentals & Hands-on Assessment

---

### 1. Linux Administration, System Operations & CLI Troubleshooting

**Q: How do you check what's consuming high CPU/memory on a server?**

```bash
top                      # live view, press 'M' for mem sort, 'P' for CPU sort
htop                     # nicer UI version
ps aux --sort=-%cpu | head -10
ps aux --sort=-%mem | head -10
vmstat 1 5                # system-wide CPU/mem/IO snapshot every 1s, 5 times
```

Answer style: "I'd start with `top`/`htop` to identify the offending PID, then `ps -p <pid> -o pid,ppid,cmd,%cpu,%mem` to inspect it, and check `/proc/<pid>/status` or `cat /proc/<pid>/limits` if I need deeper resource limit info."

**Q: How do you check disk usage and find what's filling up a disk?**

```bash
df -h                              # disk usage by filesystem
du -sh /* 2>/dev/null | sort -rh   # top-level dirs by size
du -sh /var/log/* | sort -rh       # common culprit: logs
lsof | grep deleted                # deleted-but-open files holding space (classic gotcha)
```

🔗 _Tie-in: "I hit this exact scenario with Confluent broker logs filling /var/log on bare-metal RHEL — had to set up log rotation and retention policies."_

**Q: Difference between hard link and soft (symbolic) link?**

- Hard link: another name pointing to the same inode; same data, deleting one doesn't affect the other; can't span filesystems/partitions.
- Soft link: a pointer/shortcut to a path; breaks if the target moves; can span filesystems; `ls -l` shows `->`.

```bash
ln file1 file2        # hard link
ln -s file1 file2     # soft link
```

**Q: How do you check which process is using a specific port?**

```bash
ss -tulnp | grep :8080
netstat -tulnp | grep :8080   # older systems
lsof -i :8080
```

🔗 _"I do this constantly debugging Kafka listener port collisions — e.g., resolving a 9093 port collision between controller and broker listeners during a KRaft deployment."_

**Q: Explain Linux file permissions and how to change them.**

- `rwx` for owner/group/other = read/write/execute. Numeric: r=4, w=2, x=1.

```bash
chmod 755 file.sh      # rwxr-xr-x
chmod u+x file.sh      # add execute for owner only
chown user:group file  # change ownership
chmod -R 644 /path     # recursive
```

Special bits: SUID (4000), SGID (2000), Sticky bit (1000, e.g. on `/tmp`).

**Q: systemd basics — how do you manage a service?**

```bash
systemctl status <service>
systemctl start|stop|restart|reload <service>
systemctl enable|disable <service>     # start on boot or not
journalctl -u <service> -f             # follow logs
journalctl -u <service> --since "10 min ago"
systemctl daemon-reload                # after editing unit files
```

🔗 _"I manage Confluent Kafka broker/controller services this way on bare-metal RHEL — systemd unit files, journalctl for startup failure diagnosis."_

**Q: How do you troubleshoot a server that's unreachable via SSH?**

1. Check if host is up at all: `ping <host>`
2. Check if SSH port is open: `nc -zv <host> 22` or `telnet <host> 22`
3. Check from a different network path (VPN, bastion) to rule out local firewall/routing
4. If you have console/IPMI access: check `sshd` service status, `/etc/ssh/sshd_config`, firewall rules (`iptables -L` / `firewalld`), and disk space (SSH daemon can fail to start if disk is full)
5. Check `/var/log/auth.log` or `/var/log/secure` for auth failures

**Q: What's the difference between a process and a thread? Zombie vs orphan process?**

- Process: independent execution unit with its own memory space. Thread: lightweight unit within a process sharing memory.
- **Zombie**: process finished executing but its exit status hasn't been read by the parent — entry stays in process table. Fix: signal/kill the parent so it reaps the child, or `kill -9` the parent if it's stuck.
- **Orphan**: parent died before child; child gets re-parented to `init`/`systemd` (PID 1), which reaps it automatically. Not usually a problem.

**Q: cron vs systemd timers?**

- cron: simple, widely known, `crontab -e`, minute-level granularity.
- systemd timers: better logging (journalctl), dependency management, can trigger on events not just time, easier to debug failures.

```bash
crontab -l                          # list cron jobs
0 2 * * * /path/to/script.sh        # daily at 2am
```

**Q: How do you find and kill a runaway process?**

```bash
ps aux | grep <name>
kill -15 <pid>     # graceful (SIGTERM)
kill -9 <pid>      # force (SIGKILL) — last resort
pkill -f <pattern>
```

---

### 2. Basic Networking, DNS, TCP/IP & Connectivity Analysis

**Q: Walk through what happens when you type a URL and hit enter.**

1. Browser checks DNS cache → OS cache → resolver (recursive DNS lookup: root → TLD → authoritative NS)
2. Gets IP address back
3. TCP 3-way handshake (SYN, SYN-ACK, ACK) with the server on port 443/80
4. TLS handshake (if HTTPS) — cert exchange, key negotiation
5. HTTP request sent, server responds
6. Browser renders response

**Q: TCP vs UDP — when would you use each?**

|TCP|UDP|
|---|---|
|Connection-oriented, reliable, ordered|Connectionless, no guarantee|
|Handshake + ACKs + retransmission|Fire and forget|
|Use: HTTP, databases, Kafka (TCP-based)|Use: DNS queries, video streaming, DHCP|

🔗 _"Kafka's protocol is TCP-based — I've worked extensively with broker/controller listener configs (9092, 9094, 9095 in my recent deployment) so I'm very comfortable with TCP-level troubleshooting."_

**Q: How does DNS resolution work? A record vs CNAME vs MX?**

- **A record**: hostname → IPv4 address
- **AAAA**: hostname → IPv6
- **CNAME**: alias, hostname → another hostname (can't coexist with other records at same name)
- **MX**: mail server priority records
- **TXT**: arbitrary text (SPF, domain verification)

```bash
dig example.com
dig example.com +short
nslookup example.com
dig -x 8.8.8.8           # reverse lookup
```

**Q: How do you debug "site is down" / connectivity issues?**

```bash
ping <host>                    # is it reachable at all (ICMP)?
traceroute <host>              # where does it break in the path?
dig <host>                     # is DNS resolving correctly?
curl -v https://<host>         # full request/response detail, TLS handshake info
telnet <host> <port>           # is the specific port open?
mtr <host>                     # combines ping + traceroute, live
```

Answer structure they want: **Layered troubleshooting** — L3 (ping/routing) → L4 (port/telnet) → L7 (curl/HTTP status code) → DNS as a parallel check.

**Q: What's a Load Balancer? L4 vs L7?**

- **L4 (Transport layer)**: routes based on IP/port, doesn't inspect content. Faster. (e.g., AWS NLB)
- **L7 (Application layer)**: routes based on HTTP headers/path/host. Can do SSL termination, path-based routing. (e.g., AWS ALB, NGINX Ingress) 

**Q: What's a firewall, and how do Security Groups differ from NACLs (AWS)?**

- **Security Group**: stateful, instance/ENI level, only allow rules (no explicit deny needed — return traffic auto-allowed)
- **NACL**: stateless, subnet level, allow AND deny rules, rule order matters (evaluated by rule number)

**Q: Explain subnetting / CIDR basics.**

- `/24` = 256 IPs (254 usable), `/16` = 65,536 IPs. Smaller number after `/` = more bits for network = fewer hosts.
- Public subnet: has route to Internet Gateway. Private subnet: routes through NAT Gateway for outbound only.

**Q: HTTP status codes you should know cold:**

- 200 OK, 201 Created, 301/302 redirect
- 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests
- 500 Internal Server Error, 502 Bad Gateway (upstream down/unreachable), 503 Service Unavailable (overloaded/maintenance), 504 Gateway Timeout

---

### 3. Kubernetes Fundamentals, Workload Management & Cluster Operations

**Q: Explain core Kubernetes architecture (control plane vs worker nodes).**

- **Control plane**: API Server (front door, all communication goes through it), etcd (key-value store, cluster state), Scheduler (assigns pods to nodes), Controller Manager (reconciliation loops — ReplicaSet, Node, etc.)
- **Worker nodes**: kubelet (talks to API server, manages pod lifecycle on the node), kube-proxy (networking rules), container runtime (containerd/CRI-O)

**Q: Pod vs Deployment vs ReplicaSet vs StatefulSet vs DaemonSet — when to use each?**

- **Pod**: smallest deployable unit, one or more containers sharing network/storage.
- **ReplicaSet**: ensures N replicas of a pod running. You rarely create these directly.
- **Deployment**: manages ReplicaSets, gives you rolling updates/rollbacks. Use for stateless apps.
- **StatefulSet**: stable network identity + persistent storage per pod (pod-0, pod-1...). Use for databases, Kafka, anything needing stable identity.
- **DaemonSet**: one pod per node. Use for log collectors (Fluentd), monitoring agents (node-exporter).

**Q: Explain Kubernetes Services — ClusterIP, NodePort, LoadBalancer, Ingress.**

- **ClusterIP** (default): internal-only virtual IP, reachable within cluster.
- **NodePort**: exposes service on a static port on every node's IP (30000-32767 range).
- **LoadBalancer**: provisions a cloud LB (ELB/GCP LB) pointing to the service.
- **Ingress**: L7 routing (host/path-based) into the cluster, needs an Ingress Controller (e.g., NGINX Ingress) to actually work.

**Q: What's a ConfigMap vs Secret?**

- ConfigMap: non-sensitive config data (key-value), mounted as env vars or files.
- Secret: same idea but for sensitive data — base64 encoded (NOT encrypted by default! encryption at rest must be configured separately), can integrate with external secret managers (Vault, AWS Secrets Manager).

**Q: What is a Namespace and why use it?** Logical isolation within a cluster — for multi-team/multi-env separation, resource quotas, RBAC scoping. `kubectl get pods -n <namespace>`.

**Q: Explain liveness vs readiness vs startup probes.**

- **Liveness**: "is this container alive?" — fails → kubelet restarts the container.
- **Readiness**: "is this container ready to serve traffic?" — fails → pod removed from Service endpoints (not restarted).
- **Startup**: for slow-starting apps — disables liveness/readiness checks until startup succeeds, prevents premature kills.

**Q: Common kubectl commands you should know cold:**

```bash
kubectl get pods -o wide                       # list pods with node info
kubectl describe pod <pod>                      # events, conditions, errors
kubectl logs <pod> -c <container> --previous     # logs from crashed container
kubectl logs -f <pod>                            # follow/stream logs
kubectl exec -it <pod> -- /bin/sh                # shell into pod
kubectl get events --sort-by='.lastTimestamp'    # cluster events, newest last
kubectl rollout status deployment/<name>
kubectl rollout undo deployment/<name>           # rollback
kubectl rollout history deployment/<name>
kubectl scale deployment/<name> --replicas=5
kubectl top pods                                 # needs metrics-server
kubectl get pods --field-selector=status.phase=Failed
kubectl apply -f manifest.yaml
kubectl diff -f manifest.yaml                    # preview changes before apply
```

**Q: How does a Pod get scheduled onto a node?** Scheduler filters nodes (resource requests/limits, taints/tolerations, node selectors/affinity) then scores remaining candidates and picks the best fit. Taints repel pods unless they have a matching toleration; affinity/anti-affinity attracts or spreads pods relative to other pods or node labels.

**Q: What are resource requests vs limits?**

- **Request**: guaranteed minimum, used for scheduling decisions.
- **Limit**: hard ceiling — CPU gets throttled at limit, memory over limit gets the pod OOMKilled.

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

**Q: What is Helm and why use it?** Package manager for K8s — templates manifests with values files so you can parameterize deployments across environments (dev/staging/prod) instead of maintaining raw YAML duplicates.

```bash
helm install myapp ./mychart -f values-prod.yaml
helm upgrade myapp ./mychart
helm rollback myapp 1
helm list
```

---

### 4. System Monitoring, Log Analysis & Production Troubleshooting

**Q: Explain the Grafana + Loki + Prometheus + Fluentd stack — how do they fit together?**

- **Prometheus**: metrics collection (pull-based, scrapes `/metrics` endpoints), time-series DB, alerting via Alertmanager.
- **Grafana**: visualization layer — dashboards on top of Prometheus, Loki, CloudWatch, etc.
- **Loki**: log aggregation, designed to be cheap/efficient — indexes labels not full text (unlike Elasticsearch).
- **Fluentd** (or Fluent Bit): log shipper/collector — runs as a DaemonSet, tails container logs, forwards to Loki/Elasticsearch.

Mental model: **Fluentd collects → Loki stores/indexes → Grafana visualizes; Prometheus scrapes metrics → Grafana visualizes.**

**Q: PromQL — write a query for "5xx error rate over 5 minutes."**

```promql
sum(rate(http_requests_total{status=~"5.."}[5m])) 
  / 
sum(rate(http_requests_total[5m]))
```

**Q: How do you debug a pod stuck in `CrashLoopBackOff`?**

```bash
kubectl describe pod <pod>              # check Events section at the bottom
kubectl logs <pod> --previous           # logs from the crashed instance
```

Common causes: app crashing on startup (bad config/env var), failing liveness probe killing it repeatedly, OOMKilled (check `kubectl describe` for `OOMKilled` reason), missing dependency/connection (DB not reachable).

**Q: Pod stuck in `Pending` — how do you debug?**

```bash
kubectl describe pod <pod>     # look at Events for scheduling failures
```

Common causes: insufficient cluster resources (CPU/mem), no node matches nodeSelector/affinity, PVC can't bind, taints without tolerations.

**Q: Pod stuck in `ImagePullBackOff`?** Wrong image name/tag, private registry without imagePullSecrets configured, or registry auth expired/rate-limited.

**Q: How would you set up alerting for "disk usage > 85%"?** Prometheus + node-exporter scrapes disk metrics → Alertmanager rule fires on threshold → routes to Slack/PagerDuty/email.

```yaml
- alert: HighDiskUsage
  expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100 < 15
  for: 5m
  labels:
    severity: warning
```


**Q: Difference between metrics, logs, and traces (the "three pillars of observability")?**

- Metrics: numeric, aggregated, cheap, good for trends/alerting (Prometheus).
- Logs: discrete events, detailed, more expensive to store (Loki/ELK).
- Traces: request flow across distributed services, good for latency root-causing (Jaeger/Tempo).

---

---

## ROUND 2: Advanced Technical Deep Dive

> This round tests judgment, not just recall. For every answer, structure your response as: **constraint → options → tradeoff → decision**. Interviewers are scoring _how you think_, not whether you recite the "right" answer.

---

### 1. Advanced Cloud Architecture, High Availability & Multi-Region Failover

**Q: Design a highly available 3-tier web app on AWS.** Talk through it layer by layer:

- **Network**: VPC across ≥2 AZs, public subnets (ALB, NAT GW) + private subnets (app, DB)
- **Compute**: ECS Fargate or EKS, Auto Scaling Group / HPA, spread across AZs
- **LB**: ALB in public subnets, health checks, target groups per service
- **Data**: RDS Multi-AZ (sync standby) or Aurora (multi-AZ, fast failover), read replicas for read scaling
- **Caching**: ElastiCache (Redis) for session/cache layer
- **Static/CDN**: S3 + CloudFront
- **DNS**: Route53 with health checks for failover routing

**Q: What's the difference between Multi-AZ and Multi-Region? When do you need multi-region?**

- Multi-AZ: HA within a region — protects against AZ-level failure (data center outage), low latency between AZs (single-digit ms), most apps stop here.
- Multi-Region: DR against full region failure, or latency optimization for global users, or regulatory data residency requirements. Much harder — needs data replication strategy (async usually, due to latency), DNS-based failover (Route53), and you must decide active-active vs active-passive.

**Q: Active-active vs active-passive failover — tradeoffs?**

- **Active-passive**: simpler, standby just waits, failover has some downtime (DNS propagation, promotion time), cheaper (standby can be smaller).
- **Active-active**: zero/near-zero downtime, but requires conflict resolution for writes, more complex data sync, costs 2x+ compute. Decision driver: **RTO/RPO requirements** — ask "what does the business actually tolerate?" before picking.

**Q: Explain RTO vs RPO.**

- **RTO** (Recovery Time Objective): how long can you be down? Drives your failover automation investment.
- **RPO** (Recovery Point Objective): how much data can you afford to lose? Drives your backup/replication frequency.

**Q: How do you design for "no single point of failure"?** Redundancy at every layer: multiple AZs, multiple replicas per service (never run 1 pod), LB health checks removing unhealthy targets automatically, database replicas + automated failover, stateless app tier (so any instance can die without data loss), idempotent retries for transient failures.

**Q: NAT Gateway vs NAT Instance? Why does a private subnet need NAT at all?** Private subnet has no route to Internet Gateway (that's what makes it private), but instances inside still need outbound internet (e.g., pulling OS updates, calling external APIs). NAT Gateway: AWS-managed, HA within an AZ (deploy one per AZ for full HA), no maintenance, more $$. NAT Instance: just an EC2 doing NAT — cheaper, but you manage patching/scaling/HA yourself.


# 2. Deep Kubernetes Troubleshooting, Cluster Debugging & Workload Failure Analysis

Q: A node goes `NotReady` — walk through your debugging process.

```
kubectl get nodes
kubectl describe node <node>           # check Conditions: MemoryPressure, DiskPressure, etc.
kubectl get events --field-selector involvedObject.name=<node>
```

Then SSH to the node itself (if possible):

```bash
systemctl status kubelet
journalctl -u kubelet -f
df -h          # disk pressure?
free -h        # memory pressure?
```

Common root causes: kubelet crashed/can't reach API server (network/cert issue), disk full (triggers DiskPressure, evicts pods), node-level resource exhaustion, container runtime (containerd) hung.

**Q: How do you debug intermittent 502/504 errors from your Ingress?** Layer the investigation:

1. Is it the Ingress controller itself, or upstream pods? `kubectl logs -n ingress-nginx <controller-pod>`
2. Are readiness probes flapping — pods cycling in/out of the endpoint list? `kubectl get endpoints <svc>`
3. Is it a timeout config mismatch (Ingress timeout shorter than app's actual processing time)?
4. Connection pool exhaustion — too many concurrent connections vs upstream `keepalive` settings?
5. Check if it correlates with deploys (rolling update connection draining issue) or with traffic spikes (HPA not scaling fast enough).

**Q: Explain how a rolling update actually works, and how it can go wrong.** Deployment creates new ReplicaSet, scales it up while scaling old one down, governed by `maxSurge` (how many extra pods above desired count) and `maxUnavailable` (how many can be down during rollout). **Goes wrong when**: readiness probe doesn't actually validate the app is ready (false positive → traffic hits broken pod), no `preStop` hook/graceful shutdown handling → in-flight requests dropped, `terminationGracePeriodSeconds` too short for the app to drain connections.

**Q: What causes `OOMKilled` and how do you fix it?** Container exceeded its memory `limit` → kernel OOM killer terminates it (exit code 137). Fix path: check actual usage with `kubectl top pod` over time / Grafana, right-size the limit (don't just blindly raise it — find out _why_ memory grew, could be a leak), check for missing `-Xmx` equivalent in JVM apps not respecting container limits.

**Q: etcd is critical — what happens if it goes down, and how do you protect against that?** etcd holds all cluster state. If etcd is unavailable: control plane can't schedule new things or process API calls, but **already-running pods keep running** (kubelet doesn't need etcd to keep containers alive). Protection: run etcd as an odd-numbered cluster (3 or 5 members) for quorum, regular snapshots/backups, monitor etcd disk latency (it's extremely I/O sensitive).

🔗 _"This maps very closely to KRaft controller quorum behavior I work with daily — odd-numbered controller count (I run 3: DACDCAPCTDLV01-03), quorum loss tolerance, and the same instinct that metadata-layer outage ≠ immediate data-plane outage."_

**Q: How would you debug a pod that can resolve DNS for some services but not others?** Check CoreDNS pods are healthy (`kubectl get pods -n kube-system -l k8s-app=kube-dns`), check `/etc/resolv.conf` inside the pod, verify NetworkPolicy isn't blocking egress to CoreDNS on port 53, check CoreDNS logs for errors/throttling, verify the failing service's DNS record actually exists (`kubectl get svc`).

---

### 3. Scalability Planning, Auto-Scaling & Performance Bottleneck Investigation

**Q: HPA vs VPA vs Cluster Autoscaler — explain each.**

- **HPA** (Horizontal Pod Autoscaler): adds/removes pod replicas based on metrics (CPU/mem/custom).
- **VPA** (Vertical Pod Autoscaler): adjusts resource requests/limits of existing pods (requires restart).
- **Cluster Autoscaler**: adds/removes **nodes** when pods can't be scheduled (pending due to insufficient capacity) or nodes are underutilized. They work together: HPA scales pods → if no room → Cluster Autoscaler adds nodes.

**Q: How would you find a performance bottleneck in a slow microservice?** Structured approach:

1. **Is it CPU, memory, I/O, or network bound?** → `kubectl top`, Grafana dashboards
2. **Is it the app or a dependency?** → distributed tracing (Jaeger) to see where time is spent
3. **Database?** → slow query logs, connection pool exhaustion, missing indexes
4. **Is it one pod or all pods?** → if one, it's likely a node/hardware issue; if all, it's likely code/dependency/config
5. **Correlate with deploy timeline** — did this start after a specific release?

**Q: What's a thundering herd problem, and how do you prevent it?** Many clients/processes all retry or wake up simultaneously, overwhelming a resource (e.g., cache expires, all requests hit DB at once). Prevention: jittered/exponential backoff on retries, staggered TTLs/cache warming, request coalescing (single in-flight request for the same key), circuit breakers.

**Q: Explain horizontal vs vertical scaling tradeoffs.**

- Vertical (bigger instance): simpler, no distributed-systems complexity, but has a hard ceiling and single point of failure, usually requires downtime to resize.
- Horizontal (more instances): no real ceiling, better fault tolerance, but requires the app to be stateless/distributable and adds operational complexity (LB, service discovery).

---

### 4. CI/CD Pipeline Failures, Deployment Rollbacks & Release Reliability

**Q: Your CI/CD pipeline deployment just broke production. Walk through your response.**

1. **Mitigate first, root-cause later** — roll back immediately (`kubectl rollout undo` or pipeline's automated rollback) to restore service.
2. Confirm rollback actually fixed it (don't assume).
3. **Then** investigate: diff what changed (code, config, infra-as-code, dependency version), check pipeline logs stage by stage.
4. Document timeline for postmortem.
5. Add a safeguard so this specific failure mode can't recur silently (canary, better test coverage, gate).

**Q: Blue-green vs canary vs rolling deployment — when would you choose each?**

- **Rolling** (default K8s): gradual pod-by-pod replacement, no extra infra cost, but rollback isn't instant and you have mixed versions serving traffic simultaneously.
- **Blue-green**: two full environments, switch traffic atomically, instant rollback (just switch back), but 2x infra cost during cutover.
- **Canary**: route a small % of traffic to new version, watch metrics, gradually increase. Best for catching issues with real traffic before full exposure, but needs good observability/automation to be safe.

**Q: How do you design a CI/CD pipeline from scratch for a microservice?** Stages: lint/static analysis → unit tests → build container image → push to registry (ECR) → integration tests → deploy to staging → smoke tests → manual/automated gate → deploy to prod (canary or rolling) → post-deploy health checks.

**Q: What causes flaky CI tests, and how do you deal with them?** Causes: test order dependency/shared state, timing/race conditions, external service calls without proper mocking, resource contention in parallel test runs. Approach: quarantine flaky tests (don't just ignore them — track and fix), add retries _only_ as a stopgap not a permanent fix, invest in deterministic test data/mocking.

**Q: How do you handle secrets in a CI/CD pipeline securely?** Never in code/repo. Use the CI platform's encrypted secret store (GitHub Actions Secrets, AWS Secrets Manager/Parameter Store referenced at runtime), short-lived credentials over static keys where possible (OIDC federation for GitHub Actions → AWS instead of long-lived access keys), least-privilege IAM roles scoped per pipeline.


---

### 5. Critical Thinking, Root Cause Analysis & Production Incident Handling

**Q: Walk me through your incident response process end to end.**

1. **Acknowledge & assess severity** — is this customer-impacting? How many users?
2. **Communicate early** — status page/Slack, even with "we're investigating," before you have answers.
3. **Mitigate** — stop the bleeding (rollback, scale up, failover, feature-flag off) before deep-diving root cause.
4. **Diagnose** — logs, metrics, traces, recent changes (deploys, config, infra).
5. **Resolve & verify** — confirm the fix actually worked with real signals, not assumption.
6. **Postmortem** — blameless, focused on systemic fixes (better alerting, tests, guardrails) not "who broke it."

**Q: What's the 5 Whys technique? Give an example.** Keep asking "why" until you hit a systemic root cause instead of a surface symptom. Example: Site down → Why? Pods crashed → Why? OOMKilled → Why? Memory limit too low → Why? Limit was copy-pasted from a different service's template → Why? No standard process for right-sizing resource requests → **Root cause: missing capacity-planning process**, not "we set the wrong number."

**Q: How do you prioritize during a P1 incident with multiple things going wrong simultaneously?** Triage by **blast radius × reversibility**: fix the thing causing the most customer impact first; prefer actions that are easily reversible (feature flag off, rollback) over risky ones (schema migration) under pressure; delegate — don't try to be the only person debugging, assign clear owners per workstream (comms, mitigation, root cause).

**Q: Tell me about a production issue you debugged. (Behavioral — prep a real STAR example)** Use your Confluent work — strong, concrete, technically deep:

- **Situation**: ConfluentServerAuthorizer ProvisionException on bare-metal brokers during RBAC/MDS setup, or the DN-vs-CN principal mismatch from missing `ssl.principal.mapping.rules` on controllers.
- **Task**: Get mTLS + LDAP + RBAC working correctly across a 3-controller/3-broker KRaft cluster.
- **Action**: Systematically isolated the issue — checked listener scoping (broker-only vars misplaced in controller group), validated cert CNs against `albtests.com` domain, traced the principal mapping logic.
- **Result**: Identified root cause was missing principal mapping on controllers specifically (not brokers), fixed via correct `ssl.principal.mapping.rules`, validated via successful RBAC role binding tests. This shows real depth — better than a generic story. Have 1-2 of these ready, ideally one infra/config-based and one "ambiguous/high-pressure" one if you have it.

---

---

## ROUND 3: Techno-Managerial & Practical Evaluation

> This round blends technical judgment with ownership, communication, and how you operate under pressure. Expect scenario-based questions — they want to hear your _decision-making framework_, not a textbook answer. Lead with structure, then specifics.

---

### 1. Cloud Infrastructure, High Availability & Scalability

**Q: How would you architect infrastructure for a fast-growing startup (cost vs reliability tradeoff)?** Frame it as **stage-appropriate architecture**: early stage → optimize for speed of iteration and lower cost (managed services, single-region, simpler HA); growth stage → start investing in redundancy, multi-AZ, autoscaling, better observability; scale stage → multi-region, chaos engineering, formal SLOs/error budgets. Avoid over-engineering for scale you don't have yet — that's a common trap interviewers probe for.

**Q: How do you decide between managed services (RDS, EKS) vs self-managed (EC2 + your own DB/K8s)?** Decision framework: team size/expertise (small team → managed, less ops burden), cost at scale (self-managed can be cheaper at very large scale but has hidden ops cost), compliance/control requirements (some orgs need self-managed for data residency/control), time-to-market pressure (managed = faster).

**Q: How do you think about cost optimization in cloud infra?** Right-sizing (don't over-provision — monitor actual usage and adjust), reserved instances/savings plans for predictable baseline load, spot instances for fault-tolerant/batch workloads, autoscaling to match demand instead of provisioning for peak always, S3 lifecycle policies, tagging for cost visibility/accountability per team.

---

### 2. Kubernetes, Containerization & Cluster Management

**Q: How do you manage multiple environments (dev/staging/prod) in Kubernetes cleanly?** Namespace-per-environment (within one cluster) for smaller setups, or separate clusters per environment for stronger isolation (prod blast-radius protection). Use Helm values files or Kustomize overlays to parameterize the same base manifests per environment rather than duplicating YAML.

**Q: How do you approach cluster upgrades without downtime?** Upgrade control plane first (usually has its own HA, brief API unavailability is tolerable since running pods are unaffected), then upgrade node groups in a rolling fashion — cordon, drain (respecting PodDisruptionBudgets so you don't take down all replicas of a service at once), upgrade, uncordon. Always test in staging first, and watch deprecated API versions in release notes before upgrading.

**Q: What's a PodDisruptionBudget and why does it matter operationally?** Guarantees a minimum number/percentage of pods stay available during _voluntary_ disruptions (node drains, cluster upgrades) — prevents an upgrade or autoscaler action from accidentally taking an entire service down.

---

### 3. CI/CD Automation, Release Management & Deployment Strategies

**Q: How do you build trust in a CI/CD pipeline so people aren't afraid to ship?** Strong automated test coverage at the right layers, fast feedback loops (fail fast, don't wait 40 minutes to find a lint error), progressive delivery (canary/feature flags so blast radius is small even if something's wrong), good observability tied to each release (so issues surface within minutes, not hours), easy/fast rollback as a safety net.

**Q: How do you manage release coordination across multiple teams/services?** Versioned APIs/contracts between services (don't break consumers), feature flags to decouple deploy from release, clear deprecation policy with timelines, communication via change calendars or release notes for anything with cross-team impact.

---

### 4. Security, Compliance & Access Management

**Q: How do you approach least-privilege access in a cloud environment?** Start with zero access, grant only what's needed for the role/task, use roles over individual user policies, time-bound/temporary credentials over long-lived static keys, regular access reviews to remove stale permissions, separate prod and non-prod credentials entirely.

**Q: How do you handle secrets/certificate rotation in production without downtime?** Plan rotation _before_ expiry with alerting on cert/credential age, support dual-validity windows where both old and new are briefly accepted, automate via tools like cert-manager (K8s) or Vault dynamic secrets, test rotation procedure in staging first — never let rotation be a fire-drill. 🔗 _"I work with mTLS certs daily on the Confluent side — FQDN-signed certs across DACDCAP_ nodes — rotation planning is a real, recurring concern there, not theoretical."*

**Q: What's RBAC, and how do you design role bindings sensibly (not just "give everyone admin")?** Map roles to actual job function (read-only for most engineers, write access scoped to their own namespace/service, admin restricted to a small platform team), audit role bindings periodically, prefer namespace-scoped Roles over cluster-wide ClusterRoles unless truly needed. 🔗 _"Directly aligned with the RBAC/MDS work I just did on Confluent — role bindings scoped per principal via `svc_kafka_ldap`, avoiding the trap of over-broad `rbac_component_additional_system_admins` grants."_

---

### 5. Monitoring, Incident Response & Root Cause Analysis (Managerial Lens)

**Q: How do you decide what's worth alerting on (avoiding alert fatigue)?** Alert on **symptoms that affect users** (error rate, latency, availability) not every possible internal metric. Every alert should be actionable — if there's nothing a human can/should do, it shouldn't page anyone, it should just be a dashboard. Tune thresholds based on real incident history, not guesses.

**Q: How would you build an on-call culture that doesn't burn people out?** Reasonable rotation size/frequency, clear escalation paths so one person isn't the sole bottleneck, blameless postmortems so people aren't afraid to surface issues, invest fix-time into reducing repeat pages (if the same alert fires every week, that's a backlog item, not "just deal with it").

---

### 6. Real-time Production Issue Handling, Communication & DevOps Ownership

**Q: Tell me about a time you owned a problem end-to-end, beyond just "fixing the ticket."** Use a story that shows you went from symptom → systemic fix, e.g., your Confluent debugging work where you didn't just patch one config but actually corrected a wrong assumption (the non-existent cp-ansible variable, corrected to `rbac_component_additional_system_admins`) and likely updated documentation/runbooks for the team.

**Q: How do you communicate a production incident to non-technical stakeholders?** Lead with **impact** (who/what is affected, how severely) not technical detail, give a clear next-update time even if you don't have a resolution yet ("we'll update again in 15 min"), avoid jargon — translate "OOMKilled pod in the payments namespace" into "a backend service restarted automatically and we're confirming all transactions processed correctly."

**Q: What does "DevOps ownership" mean to you, practically?** Not just "build pipelines" — it's caring about the full lifecycle: you build it, you understand its failure modes, you're the one who gets paged for it, and you proactively improve it rather than waiting for it to break. Contrast with a ticket-passing mentality — ownership means closing the loop with a permanent fix, not just resolving the symptom.

---

---

## Quick-Reference Command Cheat Sheet (keep this tab open tomorrow)

```bash
# Linux
top / htop / ps aux --sort=-%mem
df -h / du -sh /* | sort -rh
ss -tulnp / lsof -i :PORT
systemctl status|restart <svc> / journalctl -u <svc> -f
chmod / chown / ln -s

# Networking
ping / traceroute / mtr <host>
dig <host> / dig <host> +short / nslookup
curl -v https://<host>
telnet <host> <port> / nc -zv <host> <port>

# Kubernetes
kubectl get pods -o wide
kubectl describe pod <pod>
kubectl logs <pod> -f --previous
kubectl exec -it <pod> -- sh
kubectl get events --sort-by='.lastTimestamp'
kubectl rollout status|undo|history deployment/<name>
kubectl top pods/nodes
kubectl get nodes -o wide
kubectl describe node <node>

# Helm
helm install|upgrade|rollback|list

# Observability
PromQL: rate(), sum(), histogram_quantile()
journalctl -u <service> --since "10 min ago"
```

