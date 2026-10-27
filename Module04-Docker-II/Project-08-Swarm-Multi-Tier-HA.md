# Project 08 — Multi-Tier Application on Docker Swarm with Zero-Downtime Releases

| | |
|---|---|
| **Module** | 4 — Docker II |
| **Level** | L3 — Senior |
| **Role you play** | DevOps Engineer owning availability of a customer-facing service |
| **Scope** | `6-Docker-swarm.md`, `7-Service-docker-swarm.md`, `8-Deploy-multi-tier-app.md`, Compose |

---

## 1. Business problem

The customer portal runs as single containers on one host. Every release means a few minutes of
downtime, and when the host reboots the service stays down until someone notices. The business
has asked for **no visible downtime during releases** and **automatic recovery from a node or
container failure**.

You will deliver the service on a **Docker Swarm cluster**, with replicas across nodes, rolling
updates, automatic rollback on failure, and a documented release/incident runbook.

## 2. Objectives

1. Build a 3-node Swarm (1 manager + 2 workers) on Ubuntu 26.04 VMs.
2. Deploy a multi-tier stack (web → api → db) with an overlay network.
3. Run a **rolling update with measured downtime** (target: zero failed requests).
4. Prove **self-healing**: kill a container and a whole node.
5. Prove **automatic rollback** when a bad image is deployed.
6. Document the release SOP and the failure runbook.

## 3. Scope

**In scope:** `docker swarm init/join`, `docker node`, `docker service create/scale/update/ps/logs`,
`docker stack deploy/ps/services/rm`, overlay networks, replicas, placement constraints,
`update_config`/`rollback_config`, healthchecks, Compose stack files.

**Out of scope:** Kubernetes (Module 9), external load balancers, TLS termination, autoscaling.

---

## 4. Standard Operating Procedure (SOP)

> **SOP ID:** PLT-SOP-008 · **Title:** Swarm stack release and failure response

### 4.1 Cluster design

| Node | Role | Purpose |
|---|---|---|
| `swarm-mgr-1` | manager | Raft state, scheduling, `docker` CLI entrypoint |
| `swarm-wkr-1` | worker | Runs tasks |
| `swarm-wkr-2` | worker | Runs tasks |

> A single manager tolerates **zero** manager failures. Production needs 3 or 5 managers
> (quorum = `(N/2)+1`). We use one for cost, and this limitation is documented as a known risk.

### 4.2 Service design rules

1. **Stateless tiers get ≥2 replicas** so one can be replaced at a time.
2. **The database is a single replica pinned to one node** with a local volume — Swarm does not
   move volumes. (Real-world: use managed DB or shared storage.)
3. Every service has a **healthcheck**; `start_period` must exceed real start-up time.
4. Rolling updates: `parallelism: 1`, `order: start-first`, `failure_action: rollback`.
5. **Images must be pinned to an immutable tag.** `latest` makes rollback meaningless.
6. Never publish the database port.

### 4.3 Release procedure

| # | Step | Verify |
|---|---|---|
| 1 | Pre-check cluster health | `docker node ls` all `Ready/Active` |
| 2 | Record current image version | `docker service inspect --format '{{.Spec.TaskTemplate.ContainerSpec.Image}}'` |
| 3 | Start a continuous availability probe | `curl` loop logging failures |
| 4 | Deploy the new tag | `docker service update --image ...` |
| 5 | Watch convergence | `docker service ps <svc>` |
| 6 | Confirm zero failed requests | probe log |
| 7 | Stop probe, record result | release note |

### 4.4 Rollback

* **Automatic:** `failure_action: rollback` reverts when new tasks fail their healthcheck.
* **Manual:** `docker service rollback <service>` returns to the previous spec.
* **Explicit:** `docker service update --image <previous-tag> <service>`.

### 4.5 Failure response

| Event | Expected Swarm behaviour | Engineer action |
|---|---|---|
| Container crashes | Task rescheduled automatically | Confirm with `docker service ps`; investigate logs |
| Worker node down | Tasks rescheduled to healthy nodes | Check capacity; repair node; `docker node update --availability active` |
| Manager node down | **Cluster unmanageable** (single manager) | Documented risk; restore manager, consider 3 managers |
| Bad release | Auto-rollback or manual rollback | Capture logs **before** rolling back |

---

## 5. Hands-on implementation

### Step 1 — Provision three VMs

Create `swarm-mgr-1`, `swarm-wkr-1`, `swarm-wkr-2` (Ubuntu 26.04, `e2-micro`/`e2-small`,
network tag `swarm`). Install Docker CE on **all three** (Module 3 install block).

Open the Swarm ports between the nodes only:

```bash
gcloud compute firewall-rules create swarm-internal \
  --allow tcp:2377,tcp:7946,udp:7946,udp:4789 \
  --source-tags swarm --target-tags swarm

gcloud compute firewall-rules create swarm-web \
  --allow tcp:8080 --target-tags swarm --source-ranges <your-ip>/32
```

| Port | Purpose |
|---|---|
| 2377/tcp | Cluster management (managers only) |
| 7946/tcp+udp | Node discovery / gossip |
| 4789/udp | Overlay network data plane (VXLAN) |

### Step 2 — Initialise the Swarm

On the manager:

```bash
MGR_IP=$(hostname -I | awk '{print $1}')
docker swarm init --advertise-addr "$MGR_IP"
docker swarm join-token worker        # copy the printed command
```

On each worker:

```bash
docker swarm join --token <WORKER-TOKEN> <MGR_IP>:2377
```

Back on the manager:

```bash
docker node ls
docker node inspect swarm-wkr-1 --format '{{.Status.State}} {{.Spec.Availability}}'
docker info | grep -A6 Swarm
```

### Step 3 — Service fundamentals (before the stack)

```bash
docker service create --name hello --replicas 3 -p 8081:80 nginx:1.29
docker service ls
docker service ps hello                 # which node runs which task
docker service logs hello | tail
docker service scale hello=5
docker service ps hello --filter desired-state=running

# Routing mesh: ANY node IP serves the port, wherever tasks actually run
curl -sI http://<mgr-ip>:8081 | head -1
curl -sI http://<wkr1-ip>:8081 | head -1

docker service rm hello
```

### Step 4 — Build the application images

On the manager (then push to Docker Hub so workers can pull):

```bash
mkdir -p ~/swarm-stack/api && cd ~/swarm-stack/api
cat > app.py <<'EOF'
import http.server, socketserver, os, socket
VERSION = os.environ.get("APP_VERSION", "1.0.0")
class H(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200); self.send_header("Content-Type","text/plain"); self.end_headers()
        self.wfile.write(f"api v{VERSION} on {socket.gethostname()}\n".encode())
    def log_message(self, *a): pass
print("api up", flush=True)
socketserver.TCPServer(("", 5000), H).serve_forever()
EOF

cat > Dockerfile <<'EOF'
FROM python:3.12-slim
ENV APP_VERSION=1.0.0
WORKDIR /app
COPY app.py .
RUN useradd -u 10001 -m appuser && \
    apt-get update && apt-get install -y --no-install-recommends curl && \
    apt-get clean && rm -rf /var/lib/apt/lists/*
USER appuser
EXPOSE 5000
HEALTHCHECK --interval=10s --timeout=3s --start-period=10s --retries=3 \
  CMD curl -fsS http://localhost:5000/ || exit 1
CMD ["python3", "app.py"]
EOF

DH_USER=<your-dockerhub-user>
docker build -t $DH_USER/swarm-api:1.0.0 .
docker build --build-arg X=1 -t $DH_USER/swarm-api:1.1.0 \
  --label version=1.1.0 .                 # we'll override APP_VERSION at deploy time
docker login && docker push $DH_USER/swarm-api:1.0.0 && docker push $DH_USER/swarm-api:1.1.0
```

> **Important:** in Swarm, **every node pulls the image itself**. An image that exists only in
> the manager's local cache will fail to schedule on workers. Always push to a registry.

### Step 5 — The stack file

```bash
cd ~/swarm-stack
cat > stack.yml <<'EOF'
services:
  web:
    image: nginx:1.29
    ports:
      - target: 80
        published: 8080
        protocol: tcp
        mode: ingress
    configs:
      - source: nginx_conf
        target: /etc/nginx/conf.d/default.conf
    networks: [appnet]
    deploy:
      replicas: 2
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
        failure_action: rollback
      rollback_config:
        parallelism: 1
        order: stop-first
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      placement:
        constraints: [node.role == worker]

  api:
    image: ${DH_USER}/swarm-api:1.0.0
    environment:
      APP_VERSION: "1.0.0"
    networks: [appnet]
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 15s
        order: start-first
        failure_action: rollback
        monitor: 30s
      rollback_config:
        parallelism: 1
      restart_policy:
        condition: on-failure

  db:
    image: mysql:8.4
    environment:
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/db_root
      MYSQL_DATABASE: portal
    secrets: [db_root]
    volumes:
      - db_data:/var/lib/mysql
    networks: [appnet]
    deploy:
      replicas: 1
      placement:
        constraints: [node.hostname == swarm-wkr-2]   # volume lives here
      restart_policy:
        condition: on-failure

networks:
  appnet:
    driver: overlay

volumes:
  db_data:

configs:
  nginx_conf:
    file: ./nginx.conf

secrets:
  db_root:
    file: ./db_root.txt
EOF
```

```bash
cat > nginx.conf <<'EOF'
server {
    listen 80;
    location /health { return 200 "ok\n"; }
    location / {
        proxy_pass http://api:5000;
        proxy_set_header Host $host;
        proxy_connect_timeout 2s;
    }
}
EOF

# Secret file - 0600, gitignored, removed after deploy
printf '%s' '<STRONG_ROOT_PASSWORD>' > db_root.txt && chmod 600 db_root.txt
echo -e "db_root.txt\n.env" > .gitignore
```

### Step 6 — Deploy and validate

```bash
export DH_USER=<your-dockerhub-user>
docker stack deploy -c stack.yml portal

docker stack services portal
docker stack ps portal --no-trunc
docker service logs portal_api --tail 20

curl -s http://<any-node-ip>:8080/health
for i in $(seq 1 6); do curl -s http://<any-node-ip>:8080/; done   # load-balanced across tasks
```

The hostname in the response changes between calls — proof the routing mesh is balancing.

### Step 7 — Zero-downtime rolling update (the headline test)

Terminal 1 — continuous availability probe:

```bash
FAIL=0; TOTAL=0
while true; do
  TOTAL=$((TOTAL+1))
  curl -fsS --max-time 2 http://<any-node-ip>:8080/ >/dev/null || FAIL=$((FAIL+1))
  echo "$(date +%T) total=$TOTAL fail=$FAIL"
  sleep 0.5
done
```

Terminal 2 — perform the release:

```bash
docker service inspect portal_api \
  --format '{{.Spec.TaskTemplate.ContainerSpec.Image}}'     # record current

docker service update \
  --image $DH_USER/swarm-api:1.1.0 \
  --env-add APP_VERSION=1.1.0 \
  portal_api

watch -n2 'docker service ps portal_api --filter desired-state=running'
```

Confirm the new version is serving and the probe recorded **fail=0**:

```bash
for i in $(seq 1 6); do curl -s http://<any-node-ip>:8080/; done   # should show v1.1.0
docker service ps portal_api | head
```

### Step 8 — Self-healing drills

**Drill A — kill a container:**

```bash
# On the node running a task:
docker ps --filter name=portal_api -q | head -1 | xargs docker rm -f
# On the manager:
docker service ps portal_api          # old task "Failed", new task "Running"
docker service ls                     # replicas back to 3/3
```

**Drill B — drain a node (planned maintenance):**

```bash
docker node update --availability drain swarm-wkr-1
docker node ls
docker service ps portal_api          # tasks moved off wkr-1
curl -s http://<mgr-ip>:8080/health   # still serving

docker node update --availability active swarm-wkr-1
```

**Drill C — hard node failure:**

```bash
gcloud compute instances stop swarm-wkr-1 --zone <zone>
docker node ls                        # Status Down
docker service ps portal_api          # rescheduled onto wkr-2
curl -s http://<mgr-ip>:8080/         # still serving

gcloud compute instances start swarm-wkr-1 --zone <zone>
docker node ls                        # Ready again
```

### Step 9 — Bad release → automatic rollback

```bash
docker service update --image $DH_USER/swarm-api:does-not-exist portal_api

# Watch Swarm try, fail the healthcheck/pull, and revert
docker service ps portal_api --no-trunc | head
docker service inspect portal_api \
  --format '{{.UpdateStatus.State}} :: {{.UpdateStatus.Message}}'
docker service inspect portal_api \
  --format '{{.Spec.TaskTemplate.ContainerSpec.Image}}'    # back on 1.1.0
```

Manual rollback for comparison:

```bash
docker service rollback portal_api
docker service ps portal_api | head
```

---

## 6. Acceptance criteria

- [ ] `docker node ls` shows 3 nodes `Ready/Active`
- [ ] Stack deploys from a single `docker stack deploy` command
- [ ] API responses show different task hostnames (load balancing works)
- [ ] Rolling update completed with **0 failed requests** in the probe log
- [ ] Killed container is rescheduled automatically within seconds
- [ ] Stopped worker node causes no outage
- [ ] Bad image triggers automatic rollback; previous image restored
- [ ] DB password delivered via a Docker secret, not an environment variable in Git

---

## 7. Troubleshooting

| Symptom | Cause | Action |
|---|---|---|
| Task stuck in `Pending` | No node satisfies the placement constraint | `docker service ps --no-trunc`, review `constraints` |
| `No such image` on workers | Image only in the manager's local cache | Push to a registry; workers pull independently |
| Nodes `Down` straight after join | Gossip ports blocked | Open 7946/tcp+udp and 4789/udp between nodes |
| Service reachable on one node only | Overlay/VXLAN (4789/udp) blocked | Fix firewall; `docker network inspect appnet` |
| Rolling update causes dropped requests | `order: stop-first` or missing healthcheck | Use `start-first` + a real healthcheck with `start_period` |
| Update "completes" but app is broken | Healthcheck too shallow | Probe a real endpoint, not just the port |
| DB data lost after rescheduling | Local volume does not follow the task | Pin the DB with a placement constraint; use managed/shared storage in production |
| `docker stack deploy` ignores `.env` | Stack files do not auto-read `.env` like Compose | `export` the variable or render the file first |
| Manager unreachable → nothing works | Single manager, no quorum | Documented risk; run 3 managers in production |

---

## 8. Cleanup

```bash
docker stack rm portal
docker node ls
# on each worker:
docker swarm leave
# on the manager:
docker swarm leave --force
gcloud compute instances delete swarm-mgr-1 swarm-wkr-1 swarm-wkr-2 --zone <zone>
gcloud compute firewall-rules delete swarm-internal swarm-web
```

---

## 9. Deliverables

1. Repo: `stack.yml`, `nginx.conf`, `api/Dockerfile`, `api/app.py`, `.gitignore`.
2. `RELEASE-SOP-008.md` — cluster design, service rules, release steps, rollback options.
3. `FAILURE-RUNBOOK.md` — the §4.5 table with real command output from each drill.
4. Probe log proving zero failed requests during the rolling update.
5. Screenshots: `docker node ls`, `docker stack ps`, auto-rollback `UpdateStatus`.

---

## 10. Interview talking points

* *"How do you release without downtime?"* — ≥2 replicas, `start-first` rolling update,
  `parallelism: 1`, real healthchecks, and a probe measuring failed requests during the release.
* *"What happens when a node dies?"* — Swarm reschedules tasks onto healthy nodes; stateless
  tiers recover automatically, stateful ones need pinned storage or a managed service.
* *"How does automatic rollback work?"* — `update_config.failure_action: rollback` with
  `monitor`; if new tasks fail health checks within the window, Swarm reverts the spec.
* *"Why can't I just use `latest`?"* — Rollback needs a previous immutable artefact to return to.
* *"Swarm vs Kubernetes?"* — Swarm is simpler and built into Docker; Kubernetes has a far richer
  ecosystem (autoscaling, operators, ingress). Pick by team capability and requirements.
* *"What's wrong with one manager?"* — No Raft quorum; losing it makes the cluster unmanageable.
  Running tasks continue, but you cannot deploy or recover.

**CV bullet:** *Delivered a customer-facing multi-tier service on a Docker Swarm cluster with
overlay networking, secrets, healthcheck-gated rolling updates and automatic rollback; proved
zero-failed-request releases and node-failure self-healing through documented drills.*
