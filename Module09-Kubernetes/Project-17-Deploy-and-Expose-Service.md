# Project 17 — Deploy and Expose a Service on Kubernetes

| | |
|---|---|
| **Module** | 9 — Kubernetes |
| **Level** | L2 — Engineer |
| **Role you play** | Platform/DevOps Engineer onboarding the first application onto the cluster |
| **Scope** | `1-Kubernetes-installation.md`, `2-Kubernetes-deployment.md`, `3-Kubernetes-services.md`, `4-Kubernetes-ingress.md` |

---

## 1. Business problem

The company runs each microservice in a single Docker container on a dedicated VM. A VM reboot
means downtime, scaling means building another VM by hand, and every service needs its own
public IP and firewall rule. The architecture team has approved Kubernetes.

You are tasked with **standing up the cluster** and **onboarding the first two services**
(a web frontend and an API) so they are replicated, self-healing, internally discoverable, and
exposed through a **single Ingress** with hostname-based routing.

## 2. Objectives

1. Build a working kubeadm cluster (1 control plane + 2 workers) on Ubuntu 26.04.
2. Install the Calico CNI and confirm pod networking across nodes.
3. Deploy two applications with Deployments, ConfigMaps, Secrets and resource limits.
4. Expose them internally with ClusterIP Services and verify DNS-based discovery.
5. Install ingress-nginx and route `web.<domain>` and `api.<domain>` through one entry point.
6. Prove self-healing and scaling.

## 3. Scope

**In scope:** containerd + `SystemdCgroup`, kubeadm/kubelet/kubectl v1.34, Calico v3.30.0,
namespaces, Deployments, ReplicaSets, Pods, labels/selectors, ConfigMap, Secret, resource
requests/limits, Service types (ClusterIP, NodePort, LoadBalancer concept), CoreDNS discovery,
ingress-nginx controller-v1.12.1, Ingress rules, path and host routing.

**Out of scope:** rolling updates, probes, autoscaling and incident response (Project 18),
persistent storage classes, service mesh, cloud-managed clusters.

---

## 4. Standard Operating Procedure (SOP)

> **SOP ID:** K8S-SOP-017 · **Title:** Cluster build and application onboarding standard

### 4.1 Cluster build checklist

| # | Step | Verification |
|---|---|---|
| 1 | Swap disabled permanently | `swapon --show` returns nothing after reboot |
| 2 | Kernel modules + sysctl applied | `lsmod \| grep br_netfilter`, `sysctl net.ipv4.ip_forward` = 1 |
| 3 | containerd with `SystemdCgroup = true` | `systemctl status containerd` active |
| 4 | Kubernetes packages **version-pinned** | `apt-mark showhold` lists kubelet/kubeadm/kubectl |
| 5 | Pod CIDR matches the CNI config | `--pod-network-cidr=192.168.0.0/16` for Calico defaults |
| 6 | CNI installed and all nodes `Ready` | `kubectl get nodes` |
| 7 | CoreDNS pods `Running` | `kubectl -n kube-system get pods` |
| 8 | Cross-node pod connectivity tested | Ping between pods on different nodes |

### 4.2 Application onboarding rules

| # | Rule | Reason |
|---|---|---|
| 1 | One **namespace** per application/team | Blast radius, quotas, RBAC |
| 2 | Every workload sets **requests and limits** | Prevents noisy neighbours and lets the scheduler work |
| 3 | Never `latest` — pin the image tag | Reproducibility; matches the CI tag `git-<sha>` |
| 4 | Config in ConfigMaps, secrets in Secrets | No rebuilds for config changes |
| 5 | Minimum 2 replicas for anything user-facing | Survive a node/pod loss |
| 6 | Internal traffic uses **ClusterIP + DNS name** | No hardcoded IPs |
| 7 | External traffic enters via **Ingress only** | One entry point, one TLS termination point |
| 8 | Labels follow `app`, `tier`, `env` | Selectors, monitoring and cost reporting depend on them |

### 4.3 Service type decision

| Need | Use |
|---|---|
| Pod-to-pod inside the cluster | `ClusterIP` |
| Quick lab access from the node | `NodePort` (30000–32767) |
| Production HTTP/HTTPS with hostnames/TLS | `Ingress` in front of ClusterIP |
| Non-HTTP external (TCP/UDP) in cloud | `LoadBalancer` |
| Stable network IDs for stateful pods | Headless (`clusterIP: None`) |

---

## 5. Hands-on implementation

### Step 1 — Prepare all three nodes (control plane + 2 workers)

Run on **every** node:

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
sudo systemctl mask swap.target 2>/dev/null || true

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# containerd
sudo apt-get update
sudo apt-get install -y containerd apt-transport-https ca-certificates curl gnupg
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

```bash
# Kubernetes repo (v1.34)
K8S_VERSION=v1.34
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/${K8S_VERSION}/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/${K8S_VERSION}/deb/ /" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl      # pinning is mandatory
kubeadm version
```

### Step 2 — Initialise the control plane

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl get nodes          # NotReady until the CNI is installed
```

Install Calico:

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.0/manifests/custom-resources.yaml

kubectl -n calico-system get pods -w
kubectl get nodes          # control plane becomes Ready
```

Join the workers (run the printed command on each worker):

```bash
# On the control plane, regenerate if needed:
kubeadm token create --print-join-command
```

```bash
kubectl get nodes -o wide
# all three must be Ready
```

Enable completion and aliases (quality of life):

```bash
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc
source ~/.bashrc
```

### Step 3 — Prove cross-node pod networking

```bash
kubectl create ns netcheck
kubectl -n netcheck run p1 --image=nicolaka/netshoot --command -- sleep 3600
kubectl -n netcheck run p2 --image=nicolaka/netshoot --command -- sleep 3600
kubectl -n netcheck get pods -o wide        # confirm they are on DIFFERENT nodes

P2_IP=$(kubectl -n netcheck get pod p2 -o jsonpath='{.status.podIP}')
kubectl -n netcheck exec p1 -- ping -c 3 "$P2_IP"
kubectl -n netcheck exec p1 -- nslookup kubernetes.default.svc.cluster.local

kubectl delete ns netcheck
```

### Step 4 — Namespace, config and secret

`k8s/00-namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: shop
  labels:
    env: dev
    team: platform
```

`k8s/01-config.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
  namespace: shop
data:
  APP_ENV: "dev"
  LOG_LEVEL: "info"
  # The frontend talks to the API using the ClusterIP service DNS name - never an IP
  API_BASE_URL: "http://api-svc.shop.svc.cluster.local:8080"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-content
  namespace: shop
data:
  index.html: |
    <!doctype html>
    <html><head><title>ACME Shop</title></head>
    <body style="font-family:sans-serif">
      <h1>ACME Shop - Web Tier</h1>
      <p>Served by pod: <b id="h"></b></p>
      <script>document.getElementById('h').textContent = window.location.hostname;</script>
    </body></html>
```

`k8s/02-secret.yaml` (lab only — real secrets come from a secret manager):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: api-secret
  namespace: shop
type: Opaque
stringData:
  DB_USER: "appuser"
  DB_PASSWORD: "<CHANGE_ME_IN_SECRET_MANAGER>"
```

### Step 5 — Deployments

`k8s/10-api-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: shop
  labels: { app: api, tier: backend, env: dev }
spec:
  replicas: 3
  selector:
    matchLabels: { app: api }
  template:
    metadata:
      labels: { app: api, tier: backend, env: dev }
    spec:
      containers:
        - name: api
          image: hashicorp/http-echo:1.0.0
          args:
            - "-listen=:8080"
            - "-text=api ok"
          ports:
            - containerPort: 8080
              name: http
          envFrom:
            - configMapRef: { name: api-config }
            - secretRef:    { name: api-secret }
          resources:
            requests: { cpu: "50m",  memory: "64Mi" }
            limits:   { cpu: "200m", memory: "128Mi" }
          securityContext:
            allowPrivilegeEscalation: false
            runAsNonRoot: true
            runAsUser: 10001
```

`k8s/11-web-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: shop
  labels: { app: web, tier: frontend, env: dev }
spec:
  replicas: 3
  selector:
    matchLabels: { app: web }
  template:
    metadata:
      labels: { app: web, tier: frontend, env: dev }
    spec:
      containers:
        - name: web
          image: nginx:1.29
          ports:
            - containerPort: 80
              name: http
          volumeMounts:
            - name: content
              mountPath: /usr/share/nginx/html
          resources:
            requests: { cpu: "50m",  memory: "64Mi" }
            limits:   { cpu: "200m", memory: "128Mi" }
      volumes:
        - name: content
          configMap:
            name: web-content
```

### Step 6 — Services

`k8s/20-services.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-svc
  namespace: shop
spec:
  type: ClusterIP
  selector: { app: api }
  ports:
    - name: http
      port: 8080
      targetPort: http
---
apiVersion: v1
kind: Service
metadata:
  name: web-svc
  namespace: shop
spec:
  type: ClusterIP
  selector: { app: web }
  ports:
    - name: http
      port: 80
      targetPort: http
```

Apply and verify:

```bash
kubectl apply -f k8s/
kubectl -n shop get all
kubectl -n shop get endpoints          # must list pod IPs, not be empty
```

### Step 7 — Verify service discovery by DNS

```bash
kubectl -n shop run tester --rm -it --image=nicolaka/netshoot --restart=Never -- bash

# inside the pod:
nslookup api-svc.shop.svc.cluster.local
curl -s http://api-svc:8080
curl -s http://web-svc
# repeat the api curl a few times - traffic is load-balanced across endpoints
exit
```

### Step 8 — NodePort (lab access, understand why it doesn't scale)

```bash
kubectl -n shop expose deployment web --name=web-nodeport \
  --type=NodePort --port=80 --target-port=80

kubectl -n shop get svc web-nodeport
NODE_PORT=$(kubectl -n shop get svc web-nodeport -o jsonpath='{.spec.ports[0].nodePort}')
curl -s "http://<worker-ip>:${NODE_PORT}"
```

```bash
gcloud compute firewall-rules create allow-nodeport \
  --allow tcp:30000-32767 --source-ranges <your-ip>/32
```

> One port per service, no hostnames, no TLS — this is exactly why Ingress exists. Delete it:

```bash
kubectl -n shop delete svc web-nodeport
```

### Step 9 — Install ingress-nginx

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.1/deploy/static/provider/baremetal/deploy.yaml

kubectl -n ingress-nginx wait --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller --timeout=180s

kubectl -n ingress-nginx get svc ingress-nginx-controller
kubectl get ingressclass
```

### Step 10 — Ingress with host-based routing

`k8s/30-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop-ingress
  namespace: shop
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  ingressClassName: nginx
  rules:
    - host: web.shop.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-svc
                port:
                  number: 80
    - host: api.shop.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 8080
```

Path-based alternative (same host, different paths):

```yaml
    - host: shop.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: web-svc, port: { number: 80 } } }
          - path: /api
            pathType: Prefix
            backend: { service: { name: api-svc, port: { number: 8080 } } }
```

```bash
kubectl apply -f k8s/30-ingress.yaml
kubectl -n shop get ingress
kubectl -n shop describe ingress shop-ingress
```

Test (bare-metal controller uses a NodePort):

```bash
HTTP_NODEPORT=$(kubectl -n ingress-nginx get svc ingress-nginx-controller \
  -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}')
NODE_IP=<any-node-ip>

curl -s -H "Host: web.shop.local" "http://${NODE_IP}:${HTTP_NODEPORT}/"
curl -s -H "Host: api.shop.local" "http://${NODE_IP}:${HTTP_NODEPORT}/"
```

Or add to your workstation `/etc/hosts`:

```
<node-ip>  web.shop.local api.shop.local
```

### Step 11 — Prove self-healing and scaling

```bash
# Self-healing: kill a pod, the ReplicaSet replaces it
kubectl -n shop get pods -l app=api -o wide
kubectl -n shop delete pod -l app=api --field-selector=status.phase=Running --wait=false
kubectl -n shop get pods -l app=api -w     # a new pod appears

# Scaling
kubectl -n shop scale deployment api --replicas=6
kubectl -n shop get pods -l app=api -o wide     # spread across nodes
kubectl -n shop get endpoints api-svc           # endpoints grow automatically

kubectl -n shop scale deployment api --replicas=3
```

Node failure drill:

```bash
# Drain one worker
kubectl drain <worker-2> --ignore-daemonsets --delete-emptydir-data
kubectl get nodes
kubectl -n shop get pods -o wide        # pods rescheduled onto the remaining nodes
curl -s -H "Host: web.shop.local" "http://${NODE_IP}:${HTTP_NODEPORT}/"   # still serves

kubectl uncordon <worker-2>
```

---

## 6. Acceptance criteria

- [ ] 3-node cluster, all nodes `Ready`, kube packages on hold
- [ ] containerd uses `SystemdCgroup = true`
- [ ] Calico installed; pods on different nodes can ping each other
- [ ] Application lives in its own namespace with env/team labels
- [ ] Every container has requests and limits and a pinned image tag
- [ ] Config from ConfigMap, credentials from Secret, no values baked into the image
- [ ] `api-svc` and `web-svc` ClusterIP services resolve by DNS and have endpoints
- [ ] ingress-nginx installed; one Ingress routes two hostnames to two services
- [ ] Deleting a pod results in automatic replacement; scaling updates endpoints
- [ ] Draining a node keeps the service available

---

## 7. Troubleshooting

| Symptom | Cause | Action |
|---|---|---|
| `kubeadm init` fails on swap | Swap still enabled | `swapoff -a` + comment the fstab entry |
| Nodes stuck `NotReady` | No CNI installed | Apply Calico; `kubectl -n calico-system get pods` |
| Kubelet crashloops after install | cgroup driver mismatch | `SystemdCgroup = true` in `/etc/containerd/config.toml`, restart containerd + kubelet |
| Pods `Pending` | No resources / no schedulable node | `kubectl describe pod` → events; check taints and requests |
| Pods `ImagePullBackOff` | Wrong tag, private registry, no credentials | `kubectl describe pod`; add an `imagePullSecret` |
| Service has no endpoints | Selector doesn't match pod labels | Compare `kubectl get svc -o yaml` selector with pod labels |
| Can reach pod IP but not the service | kube-proxy / CNI issue | `kubectl -n kube-system get pods`; check iptables mode |
| DNS name not resolving | CoreDNS unhealthy | `kubectl -n kube-system logs -l k8s-app=kube-dns` |
| Ingress returns 404 | Wrong `Host` header, missing `ingressClassName`, wrong backend port | `kubectl describe ingress`; check the controller logs |
| Ingress returns 503 | Backend service has no ready endpoints | Fix the Deployment/Service selector |
| Join token expired | Tokens last 24 h | `kubeadm token create --print-join-command` |

---

## 8. Cleanup

```bash
kubectl delete namespace shop
kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.1/deploy/static/provider/baremetal/deploy.yaml
gcloud compute firewall-rules delete allow-nodeport
# To tear the cluster down completely:
sudo kubeadm reset -f && sudo rm -rf /etc/cni/net.d $HOME/.kube
```

---

## 9. Deliverables

1. `k8s/` manifest directory, applied with a single `kubectl apply -f k8s/`.
2. `K8S-SOP-017.md` — build checklist, onboarding rules, service-type decision table.
3. Evidence: `kubectl get nodes -o wide`, `get all -n shop`, `get endpoints`, `get ingress`.
4. `curl` transcripts for host-based routing to both services.
5. Self-healing, scaling and node-drain demonstrations with before/after pod listings.

---

## 10. Interview talking points

* *"Explain Deployment → ReplicaSet → Pod."* — The Deployment declares intent and manages
  ReplicaSets; a ReplicaSet maintains the pod count; the pod is the running unit. Self-healing
  comes from the ReplicaSet controller reconciling actual vs desired.
* *"ClusterIP vs NodePort vs Ingress?"* — ClusterIP is internal-only; NodePort opens a high port
  on every node (one per service, no TLS/hostnames); Ingress is an L7 router in front of ClusterIP
  services giving host/path routing and a single TLS termination point.
* *"Why do requests and limits matter?"* — Requests drive scheduling; limits prevent one container
  starving the node. Without them the scheduler is guessing and OOM kills are unpredictable.
* *"How do two services talk to each other?"* — Through the Service DNS name
  `name.namespace.svc.cluster.local`, never a pod IP, because pod IPs change on every restart.
* *"Pods are Pending — what do you check?"* — `kubectl describe pod` events first: insufficient
  resources, taints/tolerations, node selectors, or no PV available.
* *"Why pin the Kubernetes packages?"* — An unattended `apt upgrade` can upgrade kubelet out of
  version skew with the control plane and break the node.

**CV bullet:** *Built a production-pattern kubeadm Kubernetes cluster (containerd, Calico CNI,
version-pinned v1.34) and onboarded multi-tier applications with namespaced Deployments, resource
governance, ConfigMap/Secret-driven configuration, DNS-based service discovery and consolidated
NGINX Ingress host routing, replacing one-VM-per-service deployments.*
