# CloudTrain DevOps — 22 Live Projects

A portfolio of **22 job-experience-oriented projects** (2 per module) built **only** from the
tools and topics taught in `CT_DevOps_live_Module1` … `CT_DevOps_live_Module11`.

Every project is written the way work actually arrives in a DevOps team:

* a **business problem** raised by a stakeholder,
* a **Standard Operating Procedure (SOP)** you would attach to a change ticket,
* a **hands-on implementation** you execute,
* **validation, rollback, troubleshooting and cleanup**,
* **deliverables** you can attach to a CV / talk about in an interview.

> **Platform:** all Linux work targets **Ubuntu 26.04 LTS** VMs on the GCP free tier.
> Module 7 (Selenium) is executed on **Windows**, matching the course hands-on.

---

## How to use this portfolio

1. Complete the module's hands-on docs first — the projects assume that knowledge and add nothing new.
2. Create a **separate GCP VM (or VM set)** per project; destroy it at the end (see *Cleanup*).
3. Work through the **SOP section first**, then the **Hands-on** section. Do not skip the
   pre-checks — in a real job the pre-checks are what keep you out of an incident review.
4. Record evidence as you go (screenshots, command output, config files). The
   **Deliverables** section lists exactly what to capture.
5. Push each project's artifacts to your own GitHub repo. That repo *is* your experience proof.

### Suggested effort

| Level | Meaning |
|---|---|
| **L1 — Associate** | Guided, single server, one tool. |
| **L2 — Engineer** | Multi-component, needs design decisions. |
| **L3 — Senior** | Failure handling, rollback, HA, operational ownership. |

---

## Project index

| # | Module | Project | Level |
|---|---|---|---|
| [P01](Module01-Linux-GCP/Project-01-Gold-Image-Build-SOP.md) | 1 — Linux & GCP | Standard "Gold Build" Linux VM — provision, harden, document | L1 |
| [P02](Module01-Linux-GCP/Project-02-Linux-Triage-Runbook.md) | 1 — Linux & GCP | L1 On-Call: Linux health-check & log triage runbook | L2 |
| [P03](Module02-Git/Project-03-Branching-Release-Workflow.md) | 2 — Git | Team branching strategy & release tagging workflow | L1 |
| [P04](Module02-Git/Project-04-Conflict-Recovery-Runbook.md) | 2 — Git | Merge conflict, rebase & history-recovery runbook | L2 |
| [P05](Module03-Docker-I/Project-05-Containerize-Legacy-App.md) | 3 — Docker I | Containerize a legacy web app and publish to a registry | L1 |
| [P06](Module03-Docker-I/Project-06-Base-Image-Factory.md) | 3 — Docker I | Internal base-image factory & image promotion SOP | L2 |
| [P07](Module04-Docker-II/Project-07-WordPress-Stack-Backup.md) | 4 — Docker II | WordPress + MySQL stack with volume backup & restore drill | L2 |
| [P08](Module04-Docker-II/Project-08-Swarm-Multi-Tier-HA.md) | 4 — Docker II | Multi-tier app on Docker Swarm with zero-downtime releases | L3 |
| [P09](Module05-Puppet/Project-09-Server-Baseline-Enforcement.md) | 5 — Puppet | Enforce a server security baseline with Puppet | L2 |
| [P10](Module05-Puppet/Project-10-Puppet-Webserver-Module.md) | 5 — Puppet | Reusable Puppet module for a web-server fleet | L2 |
| [P11](Module06-Ansible/Project-11-App-Deployment-Automation.md) | 6 — Ansible | Zero-touch Tomcat application deployment with Ansible | L2 |
| [P12](Module06-Ansible/Project-12-Patching-and-Drift-Control.md) | 6 — Ansible | Monthly patching & configuration-drift control | L3 |
| [P13](Module07-Selenium/Project-13-Regression-Suite.md) | 7 — Selenium | Automated regression suite with Selenium + TestNG (Windows) | L2 |
| [P14](Module07-Selenium/Project-14-Headless-Smoke-and-Scraper.md) | 7 — Selenium | Headless smoke pack & competitor data scraper | L2 |
| [P15](Module08-Jenkins/Project-15-CI-Pipeline-Build-Test-Push.md) | 8 — Jenkins | CI pipeline: build → test → image → registry | L2 |
| [P16](Module08-Jenkins/Project-16-CD-Pipeline-Approval-Rollback.md) | 8 — Jenkins | CD pipeline with approval gate and automated rollback | L3 |
| [P17](Module09-Kubernetes/Project-17-Deploy-and-Expose-Service.md) | 9 — Kubernetes | Deploy and expose a service with Ingress | L2 |
| [P18](Module09-Kubernetes/Project-18-Day2-Ops-Runbook.md) | 9 — Kubernetes | Day-2 operations: rolling updates, probes, scaling, incidents | L3 |
| [P19](Module10-Monitoring/Project-19-Monitoring-Stack-and-Dashboards.md) | 10 — Monitoring | Infrastructure & application monitoring with Prometheus + Grafana | L2 |
| [P20](Module10-Monitoring/Project-20-Alerting-and-OnCall.md) | 10 — Monitoring | Alerting, routing to Slack and on-call runbooks | L3 |
| [P21](Module11-Terraform/Project-21-IaC-Web-Tier.md) | 11 — Terraform | Infrastructure as Code for a GCP web tier | L2 |
| [P22](Module11-Terraform/Project-22-Multi-Env-and-Drift.md) | 11 — Terraform | Multi-environment promotion and drift detection | L3 |

---

## Common prerequisites (all Linux projects)

```bash
# Ubuntu 26.04 LTS VM on GCP, non-root user with sudo
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y git curl wget vim tree unzip net-tools jq
hostnamectl                  # confirm: Ubuntu 26.04 LTS
```

GCP firewall rules are opened **per project, by tag**, never `0.0.0.0/0` on all ports:

```bash
gcloud compute firewall-rules create allow-<service> \
  --allow tcp:<port> --target-tags <tag> --source-ranges <your-ip>/32
```

---

## Evidence / grading rubric

Each project is "done" only when all five are true:

| Check | Evidence |
|---|---|
| It works | Screenshot or command output proving the acceptance criteria |
| It is repeatable | Config/code committed to Git, no undocumented manual steps |
| It is safe | Rollback procedure written **and tested at least once** |
| It is observable | You can show how you would detect it breaking |
| It is explainable | You can answer the interview questions at the end of the doc |

---

## Safety rules for every project

* Never commit credentials, service-account JSON keys, tokens or webhook URLs.
  Use placeholders such as `<TOKEN>` and keep real values in `.gitignore`d files or
  Jenkins/Ansible secret stores.
* Always run `terraform destroy` / `gcloud compute instances delete` at the end —
  free-tier credit is finite.
* Restrict firewall sources to your own IP, never the whole internet.
