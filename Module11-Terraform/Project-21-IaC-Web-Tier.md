# Project 21 — Infrastructure as Code for a GCP Web Tier

| | |
|---|---|
| **Module** | 11 — Terraform |
| **Level** | L2 — Engineer |
| **Role you play** | Cloud/DevOps Engineer replacing click-ops with reviewable infrastructure code |
| **Scope** | `Terraform-hands-on.md` — install, providers, resources, variables, outputs, state, modules |

---

## 1. Business problem

Every environment in the GCP project was built by hand in the console. Nobody can say what is
deployed, why a firewall rule exists, or who created the VM that is costing money in an unused
region. Rebuilding the web tier after an incident took two days of guesswork, and dev and
production have quietly drifted apart.

Deliver the web tier as **Terraform code**: network, firewall, instances, a load balancer and a
startup-configured NGINX — reviewable in a pull request, reproducible from scratch, and
destroyable in one command.

## 2. Objectives

1. Install Terraform from the HashiCorp apt repository on Ubuntu 26.04.
2. Authenticate to GCP correctly and pin the provider version.
3. Write a parameterised configuration — no hardcoded project IDs, regions or CIDRs.
4. Build: VPC, subnet, firewall rules, instance template, managed instance group, health check
   and a load balancer.
5. Configure the servers with `templatefile()` startup scripts.
6. Refactor the repeated parts into a reusable **module**.
7. Produce useful outputs and verify the whole stack end to end.

## 3. Scope

**In scope:** `terraform init/fmt/validate/plan/apply/destroy/output/show/state`, providers,
`required_version`/`required_providers`, resources, data sources, variables (types, defaults,
validation), locals, outputs, `count`/`for_each`, `depends_on`, `lifecycle`, `templatefile()`,
local state plus `.gitignore` hygiene, modules.

**Out of scope:** workspaces, remote backends, drift detection, import — all covered in
Project 22. Also out: Kubernetes provisioning, CI integration of Terraform.

---

## 4. Standard Operating Procedure (SOP)

> **SOP ID:** IAC-SOP-021 · **Title:** Infrastructure as Code authoring and change standard

### 4.1 The change workflow

| # | Step | Command |
|---|---|---|
| 1 | Branch from `main` | `git switch -c feat/web-tier` |
| 2 | Format | `terraform fmt -recursive` |
| 3 | Validate syntax and types | `terraform validate` |
| 4 | Generate a plan and **read every line** | `terraform plan -out=tfplan` |
| 5 | Review the plan in the pull request | attach `terraform show -no-color tfplan` |
| 6 | Peer approval required before apply | — |
| 7 | Apply the **saved plan**, not a fresh one | `terraform apply tfplan` |
| 8 | Verify outputs and a real request path | `terraform output` + `curl` |
| 9 | Merge | — |

> Applying without a saved plan means you apply something nobody reviewed.

### 4.2 Code rules

| # | Rule | Reason |
|---|---|---|
| 1 | **Pin** `required_version` and provider versions | A provider upgrade can rewrite your infrastructure |
| 2 | No hardcoded project/region/CIDR/image | The same code must build any environment |
| 3 | Every variable has a `description` and a `type` | Self-documenting; catches type errors at plan time |
| 4 | Use `validation` blocks for constrained inputs | Fail at plan time, not at apply time |
| 5 | Consistent resource naming: `<env>-<app>-<resource>` | Findable, sortable, billable |
| 6 | Label every resource with `env`, `owner`, `managed_by` | Cost allocation and ownership |
| 7 | **Never** commit `.tfstate`, `.tfvars` with secrets, or credentials | State contains secrets in plaintext |
| 8 | No secrets in variables with defaults | Use env vars or a secret manager |
| 9 | Prefer `for_each` over `count` for named resources | `count` re-indexes and destroys on list changes |
| 10 | Keep modules small and single-purpose | Reusable and reviewable |

### 4.3 Read a plan properly

| Symbol | Meaning | Attention |
|---|---|---|
| `+` | create | Normal |
| `~` | update in place | Check the attribute |
| `-/+` | **destroy and recreate** | **STOP** — is downtime acceptable? |
| `-` | destroy | **STOP** — is this intended? |
| `<=` | read (data source) | Informational |

Any `-` or `-/+` on a production resource requires explicit confirmation in the pull request.

### 4.4 Safety rules
* `terraform destroy` is **never** run against production without a change ticket and two people.
* `prevent_destroy` on anything holding data.
* Always `terraform plan` before and after a provider upgrade.
* Lock file `.terraform.lock.hcl` **is** committed — it pins provider hashes.

---

## 5. Hands-on implementation

### Step 1 — Install Terraform on Ubuntu 26.04

```bash
sudo apt-get update
sudo apt-get install -y gnupg software-properties-common curl

wget -O- https://apt.releases.hashicorp.com/gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com \
$(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt-get update
sudo apt-get install -y terraform

terraform -install-autocomplete
terraform version
```

> If the HashiCorp repo has not yet published packages for the 26.04 codename, substitute the
> previous LTS codename (`noble`) in the repo line — the binary is identical.

### Step 2 — Authenticate to GCP

```bash
# Option A: user credentials (lab)
gcloud auth application-default login
gcloud config set project <your-project-id>

# Option B: service account (how a pipeline does it)
gcloud iam service-accounts create terraform-sa \
  --display-name="Terraform automation"

gcloud projects add-iam-policy-binding <your-project-id> \
  --member="serviceAccount:terraform-sa@<your-project-id>.iam.gserviceaccount.com" \
  --role="roles/compute.admin"

gcloud projects add-iam-policy-binding <your-project-id> \
  --member="serviceAccount:terraform-sa@<your-project-id>.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser"

# Prefer impersonation over downloading a key file:
gcloud auth application-default login \
  --impersonate-service-account=terraform-sa@<your-project-id>.iam.gserviceaccount.com
```

Enable the APIs:

```bash
gcloud services enable compute.googleapis.com iam.googleapis.com
```

### Step 3 — Repository layout

```
terraform-web-tier/
├── .gitignore
├── versions.tf
├── providers.tf
├── variables.tf
├── locals.tf
├── network.tf
├── compute.tf
├── loadbalancer.tf
├── outputs.tf
├── terraform.tfvars            # non-secret values, committed
├── templates/
│   └── startup.sh.tftpl
└── modules/
    └── web-instance-group/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

`.gitignore` — get this right **before** the first apply:

```gitignore
# Never commit state - it contains plaintext secrets
*.tfstate
*.tfstate.*
.terraform/
.terraform.tfstate.lock.info

# Plan files can contain sensitive values
*.tfplan
tfplan

# Credentials
*.json
!package.json
credentials/
*secret*.tfvars
*.auto.tfvars

# DO commit the lock file:
!.terraform.lock.hcl
```

### Step 4 — Versions and provider

`versions.tf`:

```hcl
terraform {
  required_version = ">= 1.9.0, < 2.0.0"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 6.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
  }
}
```

`providers.tf`:

```hcl
provider "google" {
  project = var.project_id
  region  = var.region
  zone    = var.zone

  default_labels = {
    managed_by = "terraform"
    repository = "terraform-web-tier"
  }
}
```

### Step 5 — Variables with validation

`variables.tf`:

```hcl
variable "project_id" {
  description = "GCP project ID that owns these resources"
  type        = string

  validation {
    condition     = length(var.project_id) > 5
    error_message = "project_id must be a real GCP project ID."
  }
}

variable "region" {
  description = "GCP region for regional resources"
  type        = string
  default     = "europe-west2"
}

variable "zone" {
  description = "GCP zone for zonal resources"
  type        = string
  default     = "europe-west2-a"
}

variable "environment" {
  description = "Environment name; prefixes every resource name"
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be one of: dev, staging, prod."
  }
}

variable "app_name" {
  description = "Application name used in resource names and labels"
  type        = string
  default     = "web"
}

variable "owner" {
  description = "Team or person responsible for these resources"
  type        = string
  default     = "platform-team"
}

variable "subnet_cidr" {
  description = "CIDR range for the application subnet"
  type        = string
  default     = "10.10.0.0/24"

  validation {
    condition     = can(cidrnetmask(var.subnet_cidr))
    error_message = "subnet_cidr must be a valid IPv4 CIDR block."
  }
}

variable "admin_source_ranges" {
  description = "CIDRs allowed to reach SSH. Must NEVER be 0.0.0.0/0."
  type        = list(string)

  validation {
    condition     = !contains(var.admin_source_ranges, "0.0.0.0/0")
    error_message = "Refusing to open SSH to the entire internet. Use <your-ip>/32."
  }
}

variable "machine_type" {
  description = "Compute Engine machine type"
  type        = string
  default     = "e2-micro" # free-tier friendly
}

variable "instance_count" {
  description = "Number of web servers in the managed instance group"
  type        = number
  default     = 2

  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 10
    error_message = "instance_count must be between 1 and 10."
  }
}

variable "boot_image" {
  description = "Boot image family for the web servers"
  type        = string
  default     = "ubuntu-os-cloud/ubuntu-2604-lts-amd64"
}

variable "enable_public_ips" {
  description = "Attach ephemeral public IPs to instances (lab convenience)"
  type        = bool
  default     = false
}
```

`locals.tf`:

```hcl
locals {
  name_prefix = "${var.environment}-${var.app_name}"

  common_labels = {
    env        = var.environment
    app        = var.app_name
    owner      = var.owner
    managed_by = "terraform"
  }

  # Split "project/family" into the two arguments the data source needs
  image_parts   = split("/", var.boot_image)
  image_project = local.image_parts[0]
  image_family  = local.image_parts[1]
}
```

`terraform.tfvars`:

```hcl
project_id          = "<your-project-id>"
region              = "europe-west2"
zone                = "europe-west2-a"
environment         = "dev"
app_name            = "web"
owner               = "platform-team"
subnet_cidr         = "10.10.0.0/24"
admin_source_ranges = ["<your-ip>/32"]
machine_type        = "e2-micro"
instance_count      = 2
```

### Step 6 — Network and firewall

`network.tf`:

```hcl
resource "google_compute_network" "vpc" {
  name                            = "${local.name_prefix}-vpc"
  auto_create_subnetworks         = false
  routing_mode                    = "REGIONAL"
  delete_default_routes_on_create = false
}

resource "google_compute_subnetwork" "subnet" {
  name                     = "${local.name_prefix}-subnet"
  ip_cidr_range            = var.subnet_cidr
  region                   = var.region
  network                  = google_compute_network.vpc.id
  private_ip_google_access = true # instances without public IPs can still reach Google APIs

  log_config {
    aggregation_interval = "INTERVAL_10_MIN"
    flow_sampling        = 0.5
    metadata             = "INCLUDE_ALL_METADATA"
  }
}

# SSH: restricted source ranges only (enforced by the variable validation)
resource "google_compute_firewall" "allow_ssh" {
  name          = "${local.name_prefix}-allow-ssh"
  network       = google_compute_network.vpc.name
  description   = "SSH from approved admin ranges only"
  source_ranges = var.admin_source_ranges
  target_tags   = ["${local.name_prefix}-server"]
  priority      = 1000

  allow {
    protocol = "tcp"
    ports    = ["22"]
  }
}

# HTTP from the Google load balancer / health check ranges ONLY
resource "google_compute_firewall" "allow_lb_health" {
  name          = "${local.name_prefix}-allow-lb"
  network       = google_compute_network.vpc.name
  description   = "HTTP from Google Cloud load balancer and health-check ranges"
  source_ranges = ["130.211.0.0/22", "35.191.0.0/16"]
  target_tags   = ["${local.name_prefix}-server"]

  allow {
    protocol = "tcp"
    ports    = ["80"]
  }
}

# Internal traffic within the subnet
resource "google_compute_firewall" "allow_internal" {
  name          = "${local.name_prefix}-allow-internal"
  network       = google_compute_network.vpc.name
  source_ranges = [var.subnet_cidr]
  allow { protocol = "icmp" }
  allow {
    protocol = "tcp"
    ports    = ["0-65535"]
  }
}

# Explicit deny-all ingress at a low priority, so the intent is visible in code
resource "google_compute_firewall" "deny_all_ingress" {
  name          = "${local.name_prefix}-deny-all-ingress"
  network       = google_compute_network.vpc.name
  description   = "Default deny - documented explicitly rather than implied"
  direction     = "INGRESS"
  priority      = 65000
  source_ranges = ["0.0.0.0/0"]

  deny { protocol = "all" }

  log_config {
    metadata = "INCLUDE_ALL_METADATA"
  }
}

# NAT so private instances can reach the internet for package installs
resource "google_compute_router" "router" {
  name    = "${local.name_prefix}-router"
  region  = var.region
  network = google_compute_network.vpc.id
}

resource "google_compute_router_nat" "nat" {
  name                               = "${local.name_prefix}-nat"
  router                             = google_compute_router.router.name
  region                             = var.region
  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"

  log_config {
    enable = true
    filter = "ERRORS_ONLY"
  }
}
```

### Step 7 — The startup script template

`templates/startup.sh.tftpl`:

```bash
#!/usr/bin/env bash
# Rendered by Terraform templatefile() - do not edit on the instance.
set -euo pipefail

exec > >(tee -a /var/log/startup-script.log) 2>&1
echo "[startup] begin $(date -Is)"

export DEBIAN_FRONTEND=noninteractive
apt-get update
apt-get install -y nginx

HOSTNAME=$(hostname)
INTERNAL_IP=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/ip)
ZONE=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/zone | awk -F/ '{print $NF}')

cat > /var/www/html/index.html <<HTML
<!doctype html>
<html>
<head><title>${app_name} - ${environment}</title></head>
<body style="font-family: sans-serif; margin: 3rem;">
  <h1>${app_name} (${environment})</h1>
  <p>Provisioned by Terraform. Release: <b>${release_version}</b></p>
  <table border="1" cellpadding="6" cellspacing="0">
    <tr><td>Hostname</td><td>$${HOSTNAME}</td></tr>
    <tr><td>Internal IP</td><td>$${INTERNAL_IP}</td></tr>
    <tr><td>Zone</td><td>$${ZONE}</td></tr>
    <tr><td>Environment</td><td>${environment}</td></tr>
    <tr><td>Owner</td><td>${owner}</td></tr>
  </table>
</body>
</html>
HTML

# Health endpoint used by the load balancer health check
mkdir -p /var/www/html
echo "ok" > /var/www/html/healthz

cat > /etc/nginx/sites-available/default <<'NGINX'
server {
    listen 80 default_server;
    root /var/www/html;
    index index.html;

    location /healthz {
        access_log off;
        default_type text/plain;
        return 200 "ok\n";
    }

    location / {
        try_files $uri $uri/ =404;
    }
}
NGINX

nginx -t
systemctl enable --now nginx
systemctl restart nginx

echo "[startup] complete $(date -Is)"
```

> **Important:** inside a `.tftpl`, `${...}` is a Terraform interpolation and `$${...}` escapes to
> a literal `${...}` for the shell. Getting this wrong is the most common templatefile bug.

### Step 8 — Compute: template + managed instance group

`compute.tf`:

```hcl
data "google_compute_image" "boot" {
  family  = local.image_family
  project = local.image_project
}

resource "google_service_account" "web" {
  account_id   = "${local.name_prefix}-sa"
  display_name = "Service account for ${local.name_prefix} instances"
}

resource "google_compute_instance_template" "web" {
  name_prefix  = "${local.name_prefix}-tmpl-"
  machine_type = var.machine_type
  region       = var.region
  tags         = ["${local.name_prefix}-server"]
  labels       = local.common_labels

  disk {
    source_image = data.google_compute_image.boot.self_link
    auto_delete  = true
    boot         = true
    disk_size_gb = 10
    disk_type    = "pd-standard"
  }

  network_interface {
    subnetwork = google_compute_subnetwork.subnet.id

    dynamic "access_config" {
      for_each = var.enable_public_ips ? [1] : []
      content {
        network_tier = "STANDARD"
      }
    }
  }

  metadata = {
    enable-oslogin = "TRUE"
    startup-script = templatefile("${path.module}/templates/startup.sh.tftpl", {
      app_name        = var.app_name
      environment     = var.environment
      owner           = var.owner
      release_version = "1.0.0"
    })
  }

  service_account {
    email  = google_service_account.web.email
    scopes = ["cloud-platform"]
  }

  shielded_instance_config {
    enable_secure_boot          = true
    enable_vtpm                 = true
    enable_integrity_monitoring = true
  }

  lifecycle {
    create_before_destroy = true # a new template must exist before the old one goes
  }
}

resource "google_compute_health_check" "web" {
  name                = "${local.name_prefix}-hc"
  check_interval_sec  = 10
  timeout_sec         = 5
  healthy_threshold   = 2
  unhealthy_threshold = 3

  http_health_check {
    port         = 80
    request_path = "/healthz"
  }
}

resource "google_compute_region_instance_group_manager" "web" {
  name               = "${local.name_prefix}-mig"
  region             = var.region
  base_instance_name = "${local.name_prefix}-vm"
  target_size        = var.instance_count

  version {
    instance_template = google_compute_instance_template.web.id
  }

  named_port {
    name = "http"
    port = 80
  }

  auto_healing_policies {
    health_check      = google_compute_health_check.web.id
    initial_delay_sec = 180 # give the startup script time to install nginx
  }

  update_policy {
    type                         = "PROACTIVE"
    minimal_action               = "REPLACE"
    max_surge_fixed              = 3
    max_unavailable_fixed        = 0 # zero-downtime rollout
    instance_redistribution_type = "PROACTIVE"
  }
}
```

### Step 9 — Load balancer

`loadbalancer.tf`:

```hcl
resource "google_compute_backend_service" "web" {
  name                  = "${local.name_prefix}-backend"
  protocol              = "HTTP"
  port_name             = "http"
  load_balancing_scheme = "EXTERNAL_MANAGED"
  timeout_sec           = 30
  health_checks         = [google_compute_health_check.web.id]

  backend {
    group           = google_compute_region_instance_group_manager.web.instance_group
    balancing_mode  = "UTILIZATION"
    capacity_scaler = 1.0
  }

  log_config {
    enable      = true
    sample_rate = 1.0
  }
}

resource "google_compute_url_map" "web" {
  name            = "${local.name_prefix}-urlmap"
  default_service = google_compute_backend_service.web.id
}

resource "google_compute_target_http_proxy" "web" {
  name    = "${local.name_prefix}-proxy"
  url_map = google_compute_url_map.web.id
}

resource "google_compute_global_address" "web" {
  name = "${local.name_prefix}-ip"
}

resource "google_compute_global_forwarding_rule" "web" {
  name                  = "${local.name_prefix}-fr"
  load_balancing_scheme = "EXTERNAL_MANAGED"
  port_range            = "80"
  target                = google_compute_target_http_proxy.web.id
  ip_address            = google_compute_global_address.web.id
}
```

### Step 10 — Outputs

`outputs.tf`:

```hcl
output "load_balancer_ip" {
  description = "Public IP of the HTTP load balancer"
  value       = google_compute_global_address.web.address
}

output "application_url" {
  description = "URL to test the deployed web tier"
  value       = "http://${google_compute_global_address.web.address}"
}

output "vpc_name" {
  description = "Name of the created VPC"
  value       = google_compute_network.vpc.name
}

output "subnet_cidr" {
  description = "CIDR range of the application subnet"
  value       = google_compute_subnetwork.subnet.ip_cidr_range
}

output "instance_group" {
  description = "Self link of the regional managed instance group"
  value       = google_compute_region_instance_group_manager.web.instance_group
}

output "service_account_email" {
  description = "Service account attached to the instances"
  value       = google_service_account.web.email
}

output "admin_ssh_ranges" {
  description = "CIDRs permitted to SSH (should be your IP only)"
  value       = var.admin_source_ranges
}
```

### Step 11 — Run the workflow

```bash
terraform init
terraform fmt -recursive -check
terraform validate

terraform plan -out=tfplan
terraform show -no-color tfplan > plan-review.txt   # attach this to the pull request
less plan-review.txt

terraform apply tfplan

terraform output
terraform output -raw application_url
```

Verify:

```bash
LB_IP=$(terraform output -raw load_balancer_ip)

# The LB needs a few minutes to become healthy
for i in $(seq 1 40); do
  CODE=$(curl -s -o /dev/null -w '%{http_code}' --max-time 5 "http://${LB_IP}/") || CODE=000
  echo "attempt $i: ${CODE}"
  [ "$CODE" = "200" ] && break
  sleep 15
done

curl -s "http://${LB_IP}/" | grep -E "Hostname|Internal IP"
curl -s "http://${LB_IP}/healthz"

# Hit it repeatedly - the hostname should vary across backends
for i in $(seq 1 10); do curl -s "http://${LB_IP}/" | grep -o 'web-vm-[a-z0-9]*'; done
```

```bash
gcloud compute instance-groups list-instances \
  "$(terraform output -raw instance_group | awk -F/ '{print $NF}')" \
  --region <region>
```

### Step 12 — Prove the firewall validation works

```bash
terraform plan -var='admin_source_ranges=["0.0.0.0/0"]'
# Error: Refusing to open SSH to the entire internet. Use <your-ip>/32.
```

> This is the value of `validation` blocks: the mistake is impossible to apply, not merely
> discouraged in a wiki.

### Step 13 — Refactor into a module

`modules/web-instance-group/variables.tf`:

```hcl
variable "name_prefix"   { type = string }
variable "region"        { type = string }
variable "subnetwork_id" { type = string }
variable "machine_type"  { type = string }
variable "instance_count" {
  type    = number
  default = 2
}
variable "boot_image_self_link" { type = string }
variable "startup_script"       { type = string }
variable "service_account_email" { type = string }
variable "labels" {
  type    = map(string)
  default = {}
}
variable "network_tags" {
  type    = list(string)
  default = []
}
```

`modules/web-instance-group/main.tf` — move `google_compute_instance_template`,
`google_compute_health_check` and `google_compute_region_instance_group_manager` here, replacing
every `local.*`/`var.*` reference with the module's own variables.

`modules/web-instance-group/outputs.tf`:

```hcl
output "instance_group" {
  value = google_compute_region_instance_group_manager.web.instance_group
}

output "health_check_id" {
  value = google_compute_health_check.web.id
}
```

Call it from the root module:

```hcl
module "web_tier" {
  source = "./modules/web-instance-group"

  name_prefix           = local.name_prefix
  region                = var.region
  subnetwork_id         = google_compute_subnetwork.subnet.id
  machine_type          = var.machine_type
  instance_count        = var.instance_count
  boot_image_self_link  = data.google_compute_image.boot.self_link
  service_account_email = google_service_account.web.email
  labels                = local.common_labels
  network_tags          = ["${local.name_prefix}-server"]

  startup_script = templatefile("${path.module}/templates/startup.sh.tftpl", {
    app_name        = var.app_name
    environment     = var.environment
    owner           = var.owner
    release_version = "1.0.0"
  })
}
```

Move the existing resources into the module without destroying them:

```hcl
# moved.tf - state surgery without downtime
moved {
  from = google_compute_instance_template.web
  to   = module.web_tier.google_compute_instance_template.web
}

moved {
  from = google_compute_health_check.web
  to   = module.web_tier.google_compute_health_check.web
}

moved {
  from = google_compute_region_instance_group_manager.web
  to   = module.web_tier.google_compute_region_instance_group_manager.web
}
```

```bash
terraform init          # re-init to pick up the new module
terraform plan          # must show "0 to add, 0 to change, 0 to destroy"
```

> A refactor that shows zero changes is a **correct** refactor. If it wants to destroy and
> recreate, the `moved` blocks are wrong — fix them before applying.

### Step 14 — Inspect the state (and understand why it is sensitive)

```bash
terraform state list
terraform state show google_compute_subnetwork.subnet
terraform show -json | python3 -m json.tool | head -40

# Prove state contains sensitive material
grep -c . terraform.tfstate
```

```bash
# Confirm nothing dangerous is tracked by Git
git status --short
git check-ignore -v terraform.tfstate
```

### Step 15 — Change management drill

Scale the tier and watch the zero-downtime policy work:

```bash
terraform plan -var="instance_count=4" -out=tfplan
terraform show -no-color tfplan | grep -E '^\s+[~+-]'
terraform apply tfplan
```

Change the release version in the startup script and observe the rolling replacement:

```bash
# edit release_version = "1.1.0"
terraform plan -out=tfplan
# note: the instance template is '-/+' (recreated) but max_unavailable_fixed = 0
terraform apply tfplan

# traffic continues while instances are replaced
while true; do curl -s -o /dev/null -w '%{http_code} ' "http://${LB_IP}/"; sleep 1; done
```

---

## 6. Acceptance criteria

- [ ] Terraform installed from the HashiCorp apt repository with a keyring
- [ ] `required_version` and provider versions pinned; `.terraform.lock.hcl` committed
- [ ] No project ID, region, CIDR, image or IP hardcoded in `.tf` files
- [ ] Every variable has a type and description; constrained inputs have `validation`
- [ ] SSH open only to `<your-ip>/32`; applying `0.0.0.0/0` is rejected at plan time
- [ ] All resources labelled with `env`, `app`, `owner`, `managed_by`
- [ ] `terraform fmt -check` and `terraform validate` pass
- [ ] Startup script rendered with `templatefile()`, correct `$${}` escaping, logged to `/var/log/startup-script.log`
- [ ] Load balancer returns HTTP 200 and traffic is distributed across instances
- [ ] Auto-healing replaces a deliberately killed instance
- [ ] Refactor into a module shows **0 changes** thanks to `moved` blocks
- [ ] `.gitignore` excludes state, plans and credentials; `git check-ignore` proves it
- [ ] A reviewed plan file is applied — never a bare `terraform apply`

---

## 7. Troubleshooting

| Symptom | Cause | Action |
|---|---|---|
| `Error: could not find default credentials` | ADC not configured | `gcloud auth application-default login` |
| `Error 403: ... API has not been used` | API disabled | `gcloud services enable compute.googleapis.com` |
| `Error 409: already exists` | Resource created outside Terraform | `terraform import`, or rename (Project 22) |
| Plan shows `-/+` unexpectedly | A ForceNew attribute changed | Check which attribute; use `create_before_destroy` to avoid downtime |
| `Invalid template interpolation value` | Shell `${VAR}` not escaped in `.tftpl` | Use `$${VAR}` for shell variables |
| Instances healthy but LB returns 502 | Health-check firewall ranges missing | Allow `130.211.0.0/22` and `35.191.0.0/16` |
| LB 502 for several minutes after apply | Backends still warming up | Wait; raise `initial_delay_sec` if the startup script is slow |
| Startup script did nothing | Syntax error, or metadata key wrong | `sudo journalctl -u google-startup-scripts`; `cat /var/log/startup-script.log` |
| `e2-micro` out of quota | Free-tier limits | Reduce `instance_count` or change region |
| State file locked | A previous run crashed | `terraform force-unlock <id>` — only after confirming nothing else is running |
| Provider upgrade changes many resources | Unpinned version | Pin with `~>`; always plan after upgrading |
| Secrets visible in state | State always stores attribute values | Never commit state; encrypt the backend (Project 22) |

---

## 8. Cleanup

```bash
terraform plan -destroy -out=destroy.tfplan
terraform show -no-color destroy.tfplan | head -40   # read it
terraform apply destroy.tfplan

terraform state list        # must be empty
gcloud compute instances list
```

---

## 9. Deliverables

1. `terraform-web-tier/` repository with the full configuration and the reusable module.
2. `IAC-SOP-021.md` — change workflow, code rules, plan-symbol table, safety rules.
3. `plan-review.txt` from a real change, as it would be attached to a pull request.
4. Evidence: `terraform output`, `curl` showing load-balanced responses, auto-healing recovery.
5. Proof that the `0.0.0.0/0` SSH plan is rejected.
6. `terraform plan` output after the module refactor showing **0 to add, 0 to change, 0 to destroy**.

---

## 10. Interview talking points

* *"What is Terraform state and why does it matter?"* — The mapping between configuration and real
  resources. It drives the plan, it contains sensitive attribute values in plaintext, and it must
  never be committed to Git or edited by hand.
* *"`terraform plan` vs `apply`?"* — Plan is the dry run and the review artefact; I save it and
  apply the saved file, so what was reviewed is exactly what runs.
* *"How do you avoid downtime on a change?"* — Read the plan for `-/+`, use
  `create_before_destroy`, and set `max_unavailable_fixed = 0` on the instance-group update policy.
* *"`count` vs `for_each`?"* — `count` is positional, so removing an element re-indexes and
  destroys unrelated resources; `for_each` keys by a stable identifier.
* *"How do you make configuration reusable?"* — Modules with explicit input variables and outputs,
  no hardcoded environment values, and `moved` blocks so refactoring doesn't destroy anything.
* *"How do you keep secrets out of Terraform?"* — No secret defaults, `.gitignore` for state/tfvars,
  values from environment variables or a secret manager, and an encrypted remote backend.
* *"Why validation blocks?"* — They make unsafe configurations impossible to apply. My firewall
  variable physically refuses `0.0.0.0/0`.

**CV bullet:** *Replaced manual GCP console provisioning with peer-reviewed Terraform — pinned
providers, validated variables that make open-SSH misconfiguration impossible, templated instance
bootstrap, regional managed instance group behind a load balancer with auto-healing and
zero-downtime update policy, refactored into a reusable module with zero-change state migration.*
