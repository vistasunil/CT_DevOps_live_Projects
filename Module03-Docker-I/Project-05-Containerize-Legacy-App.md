# Project 05 — Containerize a Legacy Web Application and Publish to a Registry

| | |
|---|---|
| **Module** | 3 — Docker I |
| **Level** | L1 — Associate |
| **Role you play** | DevOps Engineer on an application-modernisation squad |
| **Scope** | `Docker-commands.md`, `Dockerfile.md`, `Docker-commit.md`, `Docker-push.md` |

---

## 1. Business problem

An internal web application runs on a hand-built Apache VM. Nobody knows exactly how it was
configured — the person who built it left. Deploying a second copy for UAT takes two days of
guesswork, and dev/test/prod have drifted apart.

Your task: **containerize the application** so that one immutable image can be run anywhere,
publish it to Docker Hub with a sane tagging scheme, and document the migration so the next
application can follow the same path.

## 2. Objectives

1. Reverse-engineer the running app into a Dockerfile.
2. Produce an image that starts clean with no manual steps.
3. Tag it properly (never rely on `latest` in deployments) and push to a registry.
4. Prove reproducibility: destroy the container, re-run from the image, identical result.
5. Show the difference between the `docker commit` (snapshot) and Dockerfile (declarative) approaches, and justify choosing the Dockerfile.

## 3. Scope

**In scope:** Docker CE install on Ubuntu 26.04, `docker run/ps/logs/exec/stop/rm/images/rmi`,
Dockerfile instructions (`FROM`, `LABEL`, `ENV`, `RUN`, `COPY`, `WORKDIR`, `EXPOSE`, `CMD`,
`ENTRYPOINT`), `docker commit`, `docker tag`, `docker login`, `docker push`.

**Out of scope:** Compose, volumes, networks, Swarm (Module 4); orchestration (Module 9).

---

## 4. Standard Operating Procedure (SOP)

> **SOP ID:** CNT-SOP-005 · **Title:** Containerize an existing application and publish an image

### 4.1 Discovery checklist (do this before writing a single line of Dockerfile)

| # | Question | How to answer on the legacy host |
|---|---|---|
| 1 | What runtime/packages? | `dpkg -l`, `apache2 -v` |
| 2 | What ports? | `sudo ss -tulpn` |
| 3 | Where is the content/config? | `/etc/apache2/`, `/var/www/html` |
| 4 | What user does it run as? | `ps -eo user,comm | grep apache` |
| 5 | What environment variables? | `/etc/environment`, service unit file |
| 6 | What is persistent state? | DB files, uploads, logs |
| 7 | How is it started? | `systemctl cat apache2` |

### 4.2 Image build standards

1. **Pin the base image tag** (`httpd:2.4`, not `httpd:latest`) — reproducible builds.
2. **One concern per image.** Web server and database are separate images.
3. **`RUN` commands are chained and clean up** (`apt-get clean && rm -rf /var/lib/apt/lists/*`) to keep layers small.
4. **`DEBIAN_FRONTEND=noninteractive`** for any `apt-get install` in a build.
5. **Exec-form `CMD`/`ENTRYPOINT`** (`["nginx","-g","daemon off;"]`) so signals reach PID 1.
6. **Never bake secrets into an image.** Anyone who pulls it can read every layer.
7. **Label every image** with source repo, version and maintainer.
8. **The container must log to stdout/stderr**, not to a file inside the container.

### 4.3 Tagging standard

| Tag | Meaning |
|---|---|
| `<repo>:1.2.3` | Immutable release — what deployments reference |
| `<repo>:1.2` | Moves with patch releases |
| `<repo>:git-<short-sha>` | Traceability back to source |
| `<repo>:latest` | Convenience for developers only — **never** in a deployment |

### 4.4 Rollback
Deployments reference an immutable version tag, so rollback = run the previous tag.
Never delete a published tag that has been deployed.

---

## 5. Hands-on implementation

### Step 1 — Install Docker CE on Ubuntu 26.04 LTS

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin

sudo usermod -aG docker $USER && newgrp docker
docker version && docker info | head -20
```

### Step 2 — Discovery on the "legacy" server

Simulate the legacy host (or use a real one):

```bash
sudo apt-get install -y apache2
echo "<h1>Legacy Payments Portal v1</h1>" | sudo tee /var/www/html/index.html
sudo systemctl enable --now apache2

# Discovery - record all of this in the migration doc
apache2 -v
sudo ss -tulpn | grep ':80'
ls -l /var/www/html/
systemctl cat apache2 | head -20
ps -eo user,comm | grep -i apache
```

### Step 3 — Approach A: `docker commit` (snapshot) — understand why it is wrong

```bash
docker run -dit --name legacy-snap -p 8081:80 httpd:2.4
docker exec -it legacy-snap bash
# inside:
#   echo "<h1>Legacy Payments Portal v1</h1>" > /usr/local/apache2/htdocs/index.html
#   exit

docker commit -m "snapshot of manually configured portal" -a "Your Name" \
  legacy-snap portal-snapshot:0.1

docker images | grep portal
docker history portal-snapshot:0.1
```

> **Why this is rejected for production:** the image records *what* changed but not *how* or
> *why*. It is not reviewable, not diffable in Git, and not reproducible. `docker commit` is a
> legitimate tool for **capturing a broken container for debugging** — not for building releases.

### Step 4 — Approach B: the Dockerfile (what you actually deliver)

```bash
mkdir -p ~/portal-container/site && cd ~/portal-container
cat > site/index.html <<'EOF'
<!doctype html>
<html>
  <head><title>Payments Portal</title></head>
  <body>
    <h1>Payments Portal</h1>
    <p>Containerized build. Version: <span id="v">1.0.0</span></p>
  </body>
</html>
EOF
```

```bash
cat > Dockerfile <<'EOF'
# Pinned base - reproducible builds
FROM httpd:2.4

LABEL org.opencontainers.image.title="payments-portal" \
      org.opencontainers.image.version="1.0.0" \
      org.opencontainers.image.source="https://github.com/<you>/portal-container" \
      maintainer="you@example.com"

ENV APP_ENV=production \
    TZ=Asia/Kolkata

# Single cleaned-up layer; noninteractive prevents build hangs
RUN export DEBIAN_FRONTEND=noninteractive && \
    apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /usr/local/apache2/htdocs
COPY site/ ./

EXPOSE 80

HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -fsS http://localhost/ > /dev/null || exit 1

# Exec form: httpd becomes PID 1 and receives SIGTERM correctly
CMD ["httpd-foreground"]
EOF
```

```bash
cat > .dockerignore <<'EOF'
.git
*.md
*.log
.env
EOF
```

### Step 5 — Build, inspect, run

```bash
docker build -t payments-portal:1.0.0 .

docker images | grep payments-portal
docker history payments-portal:1.0.0        # see every layer and its size
docker inspect payments-portal:1.0.0 | jq '.[0].Config.Labels'

docker run -d --name portal -p 8080:80 payments-portal:1.0.0

docker ps
docker logs portal
curl -s http://localhost:8080 | head -5
docker inspect --format '{{.State.Health.Status}}' portal
```

### Step 6 — Operate the container (the commands you use daily)

```bash
docker exec -it portal bash          # shell inside
docker top portal                    # processes
docker stats --no-stream portal      # live resource usage
docker port portal                   # port mapping
docker logs -f --tail 20 portal      # follow logs
docker stop portal && docker start portal
docker rm -f portal
```

### Step 7 — Prove reproducibility

```bash
docker rm -f portal
docker run -d --name portal -p 8080:80 payments-portal:1.0.0
curl -s http://localhost:8080 | grep -o "Payments Portal"
```

Same image → same result, every time, on any host. That is the deliverable.

### Step 8 — Tag and push to Docker Hub

```bash
docker login                       # use a Personal Access Token, not your password

DH_USER=<your-dockerhub-user>
GIT_SHA=$(git rev-parse --short HEAD 2>/dev/null || echo manual)

docker tag payments-portal:1.0.0 $DH_USER/payments-portal:1.0.0
docker tag payments-portal:1.0.0 $DH_USER/payments-portal:1.0
docker tag payments-portal:1.0.0 $DH_USER/payments-portal:git-$GIT_SHA
docker tag payments-portal:1.0.0 $DH_USER/payments-portal:latest

docker push $DH_USER/payments-portal:1.0.0
docker push $DH_USER/payments-portal:1.0
docker push $DH_USER/payments-portal:git-$GIT_SHA
docker push $DH_USER/payments-portal:latest

docker images | grep payments-portal
```

### Step 9 — Pull-and-run test from a *clean* machine

```bash
# On a second VM with Docker installed, nothing cached:
docker rmi -f $DH_USER/payments-portal:1.0.0 2>/dev/null
docker run -d --name portal-verify -p 8080:80 $DH_USER/payments-portal:1.0.0
curl -s http://localhost:8080 | grep -o "Payments Portal"
```

### Step 10 — Ship a change (v1.1.0)

```bash
sed -i 's/1.0.0/1.1.0/' site/index.html
docker build -t payments-portal:1.1.0 .
docker tag payments-portal:1.1.0 $DH_USER/payments-portal:1.1.0
docker push $DH_USER/payments-portal:1.1.0

# Rollback is simply running the previous immutable tag:
docker rm -f portal
docker run -d --name portal -p 8080:80 $DH_USER/payments-portal:1.0.0
```

---

## 6. Acceptance criteria

- [ ] Dockerfile builds with no warnings and no interactive prompts
- [ ] Base image is version-pinned; image carries OCI labels
- [ ] Container serves the app on first run with **zero** manual steps
- [ ] `docker inspect ... .State.Health.Status` reports `healthy`
- [ ] Image pushed with at least 3 tags including an immutable version tag
- [ ] Pull-and-run verified on a second machine
- [ ] No secrets anywhere in `docker history`
- [ ] You can explain `commit` vs Dockerfile and `CMD` vs `ENTRYPOINT`

---

## 7. Troubleshooting

| Symptom | Cause | Action |
|---|---|---|
| `permission denied ... docker.sock` | User not in `docker` group | `sudo usermod -aG docker $USER` then re-login |
| Build hangs on `apt-get install` | Interactive prompt | `ENV DEBIAN_FRONTEND=noninteractive` / `-y` |
| Container exits immediately | Main process daemonized | Run in foreground (`httpd-foreground`, `nginx -g 'daemon off;'`) |
| `docker stop` takes 10s | Shell-form CMD; PID 1 is `/bin/sh` | Use exec form `CMD ["..."]` |
| Changes to site not visible | Rebuilt image but reused old container | `docker rm -f` then run the new tag |
| Huge image | Separate `RUN` layers, apt cache kept | Chain `RUN`, `apt-get clean`, `rm -rf /var/lib/apt/lists/*` |
| `denied: requested access to the resource is denied` on push | Image not namespaced with your user | `docker tag local <user>/<repo>:<tag>` |
| Port already allocated | Host port in use | `ss -tulpn | grep <port>` and pick another |

---

## 8. Cleanup

```bash
docker rm -f portal portal-verify legacy-snap 2>/dev/null
docker rmi payments-portal:1.0.0 payments-portal:1.1.0 portal-snapshot:0.1 2>/dev/null
docker system df && docker system prune -f
```

---

## 9. Deliverables

1. Git repo: `Dockerfile`, `.dockerignore`, `site/`, `README.md`.
2. `MIGRATION-REPORT.md` — discovery table, decisions, before/after deployment time.
3. Public Docker Hub repository with the tag list visible.
4. `IMAGE-STANDARD.md` — build rules and tagging scheme from §4.
5. Screenshots: `docker build` output, `docker images`, app in a browser, Docker Hub tags page.

---

## 10. Interview talking points

* *"How would you containerize an existing application?"* — Discovery checklist first, then a
  Dockerfile, then prove reproducibility, then registry + tagging.
* *"`docker commit` or Dockerfile?"* — Dockerfile: reviewable, version-controlled, reproducible.
  `commit` is a debugging tool.
* *"`CMD` vs `ENTRYPOINT`?"* — `ENTRYPOINT` = the fixed executable, `CMD` = default arguments;
  exec form so PID 1 receives signals.
* *"Why not use `latest` in production?"* — It is mutable; two deploys of "latest" can be
  different code, making rollback and incident analysis impossible.
* *"How do you keep images small and safe?"* — Pinned slim base, chained `RUN` with cache clean,
  `.dockerignore`, no secrets (they survive in layers), non-root where possible.

**CV bullet:** *Containerized a legacy Apache-based application into a reproducible, labelled,
health-checked Docker image with an immutable semantic-version tagging scheme published to a
registry, replacing a two-day manual server build with a one-command deployment.*
