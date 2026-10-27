# Project 01 — Standard "Gold Build" Linux VM

| | |
|---|---|
| **Module** | 1 — Linux Basics & GCP Free Tier |
| **Level** | L1 — Associate |
| **Role you play** | Junior Linux / Cloud Engineer, Infrastructure Build team |
| **Scope** | GCP free-tier VM provisioning, Linux filesystem, users & groups, permissions, packages, services, networking commands |

---

## 1. Business problem

Your company spins up Linux servers on demand, but every engineer builds them differently:
different packages, different users, inconsistent timezones, no standard log locations. When an
incident happens, nobody knows what "normal" looks like on a given box.

The Infrastructure Lead has asked you to define and build a **Gold Build** — a documented,
repeatable Ubuntu 26.04 LTS server baseline that every new VM must match — and to produce the
**SOP that a new joiner can follow to build one in under 30 minutes**.

## 2. Objectives

1. Provision an Ubuntu 26.04 LTS VM on the GCP free tier.
2. Apply a documented baseline: users, groups, permissions, packages, timezone, services.
3. Produce a **build verification checklist** that proves the server matches the standard.
4. Hand over an SOP that another engineer can execute without asking you questions.

## 3. Scope

**In scope:** everything in `CT_DevOps_live_Module1` — `GCP-free-tier-setup.md` and
`Linux-basic-commands.md` (navigation, files, permissions, users, `apt`/`dpkg`, `systemctl`,
`journalctl`, `ss`, `ip`, disk and process commands).

**Out of scope:** configuration management (Module 5/6), containers, Terraform. The whole point
of this project is to feel the manual pain that later modules automate.

---

## 4. Standard Operating Procedure (SOP)

> **SOP ID:** INFRA-SOP-001 · **Title:** Build a standard Ubuntu 26.04 LTS server
> **Owner:** Infrastructure Build team · **Review cycle:** every 6 months

### 4.1 Purpose
Deliver a consistent, auditable Linux server baseline so that monitoring, access control and
troubleshooting behave identically on every host.

### 4.2 When to use
* New application server request
* Rebuilding a server after a failure
* Creating a template/base for a new environment

### 4.3 Pre-requisites and pre-checks

| # | Pre-check | Pass criteria |
|---|---|---|
| 1 | GCP billing account active, free-tier project selected | `gcloud config get-value project` returns the right project |
| 2 | Compute Engine API enabled | VM creation page loads without an "enable API" prompt |
| 3 | Requester has supplied hostname, environment, owner, purpose | Ticket fields complete |
| 4 | Free-tier quota available | Only `e2-micro` in a free-tier region, 1 VM per project |

### 4.4 Approvals
| Change type | Approver |
|---|---|
| New non-prod VM | Team lead |
| New prod VM | Team lead + Change Advisory Board |

### 4.5 Execution steps (summary — detail in §5)
1. Create the VM with the agreed machine type, region and network tags.
2. Connect over SSH and confirm the OS release.
3. Apply OS updates.
4. Set hostname, timezone and locale.
5. Create the application user, group and standard directory tree with correct permissions.
6. Install the baseline package set.
7. Enable and verify baseline services.
8. Record the network posture (listening ports, routes).
9. Run the build verification checklist (§6) and attach the output to the ticket.

### 4.6 Rollback
The VM is disposable at this stage. If the build fails verification, **delete and rebuild** —
do not hand-patch a half-built server:
```bash
gcloud compute instances delete <vm-name> --zone <zone>
```

### 4.7 Post-implementation
* Attach verification output to the ticket.
* Add the host to the asset inventory (hostname, IP, owner, purpose, build date).
* Close the ticket only after the requester confirms SSH access.

---

## 5. Hands-on implementation

### Step 1 — Provision the VM

Console: **Compute Engine → VM instances → Create instance**

| Field | Value |
|---|---|
| Name | `gold-build-01` |
| Region / Zone | a free-tier eligible region (e.g. `us-central1` / `us-central1-a`) |
| Machine type | `e2-micro` |
| Boot disk | Ubuntu 26.04 LTS, 30 GB standard persistent disk |
| Firewall | leave HTTP/HTTPS **unchecked** — we add tagged rules only when needed |
| Network tags | `gold-build` |

### Step 2 — Connect and identify the system

```bash
ssh <user>@<external-ip>          # or use the browser SSH button

hostnamectl                       # OS, kernel, virtualization
uname -a
cat /etc/os-release
uptime
whoami && id
```

Record the output — this is your "before" state.

### Step 3 — Patch the OS

```bash
sudo apt-get update
sudo apt-get -y upgrade
sudo apt-get -y autoremove

# Did the kernel change? If yes a reboot is required.
[ -f /var/run/reboot-required ] && echo "REBOOT REQUIRED" && sudo reboot
```

### Step 4 — Identity of the host

```bash
sudo hostnamectl set-hostname gold-build-01
sudo timedatectl set-timezone Asia/Kolkata      # use your team standard
timedatectl
```

Add the hostname to `/etc/hosts` so `sudo` never complains:

```bash
sudo vim /etc/hosts
# 127.0.1.1   gold-build-01
```

### Step 5 — Users, groups and permissions (the baseline)

```bash
# Application service account - no interactive login
sudo groupadd -f appgroup
sudo useradd -m -s /usr/sbin/nologin -g appgroup appuser

# A human operator who may use sudo
sudo useradd -m -s /bin/bash -G sudo,appgroup opsuser
sudo passwd opsuser

# Standard directory tree
sudo mkdir -p /opt/app/{bin,conf,logs,data}
sudo chown -R appuser:appgroup /opt/app
sudo chmod 750 /opt/app
sudo chmod 770 /opt/app/logs          # app writes, group reads
sudo chmod 640 /opt/app/conf          # config not world-readable

# setgid so new files inherit the group
sudo chmod g+s /opt/app/logs

ls -ld /opt/app /opt/app/*
id appuser && id opsuser
```

> **Why it matters at work:** 90% of "permission denied" incidents come from an app running as
> one user while the directory is owned by another. Writing the ownership model down *before*
> deployment is the whole job here.

### Step 6 — Baseline package set

```bash
sudo apt-get install -y \
  vim curl wget git unzip tree jq \
  net-tools iproute2 htop sysstat \
  ca-certificates gnupg lsb-release

# Verify what you actually installed
dpkg -l | grep -E '^ii\s+(vim|curl|git|jq)\s'
apt list --installed 2>/dev/null | wc -l
```

### Step 7 — Services

```bash
# Confirm core services are enabled and healthy
systemctl is-enabled ssh && systemctl is-active ssh
systemctl status ssh --no-pager

# List everything that starts at boot - this is part of the baseline record
systemctl list-unit-files --state=enabled --no-pager

# Anything failed?
systemctl --failed --no-pager
```

### Step 8 — Network posture

```bash
ip addr show
ip route show
ss -tulpn                      # which processes listen on which ports
sudo ss -tulpn | grep LISTEN
```

Document every listening port and why it is open. An undocumented listener is a finding in an
audit.

### Step 9 — Logs and disk

```bash
journalctl -b --no-pager | tail -50       # this boot
journalctl -p err -b --no-pager           # errors only
journalctl -u ssh --since "1 hour ago" --no-pager

df -h
du -sh /var/log
free -h
lsblk
```

---

## 6. Build verification checklist

Run this and attach the output to the ticket.

```bash
cat <<'EOF' > /tmp/verify-gold-build.sh
#!/usr/bin/env bash
echo "=== GOLD BUILD VERIFICATION : $(date) ==="
echo "--- OS ---";            . /etc/os-release; echo "$PRETTY_NAME"
echo "--- Hostname ---";      hostnamectl --static
echo "--- Timezone ---";      timedatectl show -p Timezone --value
echo "--- Pending updates ---"; apt-get -s upgrade | grep -c '^Inst' || true
echo "--- Users ---";         id appuser; id opsuser
echo "--- App tree ---";      ls -ld /opt/app /opt/app/bin /opt/app/conf /opt/app/logs /opt/app/data
echo "--- Failed units ---";  systemctl --failed --no-pager
echo "--- Listening ports ---"; ss -tulpn | grep -i listen
echo "--- Disk ---";          df -h /
echo "=== END ==="
EOF
chmod +x /tmp/verify-gold-build.sh
sudo /tmp/verify-gold-build.sh | tee ~/gold-build-verification.txt
```

### Acceptance criteria

- [ ] `PRETTY_NAME` reports Ubuntu 26.04 LTS
- [ ] `apt-get -s upgrade` shows **0** pending packages
- [ ] Hostname and timezone match the standard
- [ ] `appuser` exists with `nologin` shell; `opsuser` is in `sudo`
- [ ] `/opt/app` tree exists with the documented ownership and modes
- [ ] `systemctl --failed` is empty
- [ ] Every listening port is documented with a justification

---

## 7. Troubleshooting

| Symptom | Likely cause | Action |
|---|---|---|
| `sudo: unable to resolve host` | Hostname not in `/etc/hosts` | Add `127.0.1.1 <hostname>` |
| `Permission denied` writing to `/opt/app/logs` | Process runs as a different user | `ps -eo user,comm | grep <app>` then fix ownership |
| `apt` fails with "Could not get lock" | Unattended-upgrades is running | `sudo systemctl status unattended-upgrades`, wait or stop it |
| SSH times out after reboot | Ephemeral external IP changed | Re-read the IP from the console, or reserve a static IP |
| `ss`/`netstat` not found | Minimal image | `sudo apt-get install -y iproute2 net-tools` |

---

## 8. Cleanup

```bash
gcloud compute instances delete gold-build-01 --zone <zone>
gcloud compute firewall-rules delete allow-gold-build   # if created
```

---

## 9. Deliverables (what goes in your portfolio)

1. `SOP-INFRA-001-gold-build.md` — the SOP from §4 in your own words.
2. `verify-gold-build.sh` — the verification script.
3. `gold-build-verification.txt` — a real passing run.
4. `baseline-inventory.md` — table of users, directories/modes, packages, enabled services, open ports.
5. Screenshots: GCP VM creation page, successful SSH session, verification output.

---

## 10. Interview talking points

* *"Walk me through how you build a new Linux server."* — Give the SOP: pre-checks, build,
  verify, record, hand over.
* *"How do you decide file permissions for an application?"* — Service account with `nologin`,
  group ownership, `750`/`770` with setgid on log dirs, config not world-readable.
* *"A teammate says a port is open that shouldn't be — how do you check?"* — `sudo ss -tulpn`,
  map PID to process, compare against the documented baseline.
* *"Why document a baseline at all?"* — Faster incident triage, audit evidence, and it is the
  input to automating the same build in Puppet/Ansible/Terraform later.

**CV bullet:** *Defined and documented a standard Ubuntu 26.04 LTS server baseline (users,
permissions, package set, services, network posture) with an automated verification script,
reducing new-server build time and eliminating configuration inconsistency between hosts.*
