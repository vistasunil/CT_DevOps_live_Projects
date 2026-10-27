# Project 12 — Monthly Patching & Configuration-Drift Control with Ansible

| | |
|---|---|
| **Module** | 6 — Ansible |
| **Level** | L3 — Senior |
| **Role you play** | Automation Engineer owning the estate's patch and compliance cycle |
| **Scope** | `Ansible-hands-on.md` — inventory, playbooks, roles, facts, `--check`, handlers, reporting |

---

## 1. Business problem

Security requires every Linux server to be patched monthly and to pass a configuration audit.
Today it is a spreadsheet, four engineers and a weekend. Servers are missed, reboots happen
unannounced, and the audit evidence is screenshots pasted into a document.

Deliver an **automated patch cycle** with pre-flight checks, staged waves, controlled reboots,
post-patch verification and a **machine-generated compliance report** — plus a drift detection
run that proves the estate matches the baseline.

## 2. Objectives

1. Build a patching playbook with pre-checks, staged waves and reboot control.
2. Only reboot hosts that **need** a reboot, during the approved window.
3. Verify service health after every patch wave; stop the cycle on failure.
4. Build a **drift-detection** play using `--check` that reports without changing anything.
5. Generate an HTML/CSV compliance report from Ansible facts.

## 3. Scope

**In scope:** `ansible.builtin.apt` (upgrade, autoremove), `ansible.builtin.reboot`,
`wait_for_connection`, `setup`/facts, `serial`, `any_errors_fatal`, `--check --diff`, `assert`,
`uri`, `stat`, `template` (for reports), `delegate_to: localhost`, `run_once`, handlers, tags.

**Out of scope:** CVE scanners, Satellite/Landscape, kernel live patching, CI (Module 8).

---

## 4. Standard Operating Procedure (SOP)

> **SOP ID:** SEC-SOP-012 · **Title:** Monthly OS patch cycle and drift control

### 4.1 Patch waves

| Wave | Hosts | Window | Approval | Gate to next wave |
|---|---|---|---|---|
| 0 — canary | 1 non-prod host | Any time | Team lead | 24 h with no issues |
| 1 — non-prod | All dev/test | Sat 10:00–14:00 | Team lead | All health checks pass |
| 2 — prod batch A | 50% of prod | Sat 22:00–02:00 | CAB | All health checks pass |
| 3 — prod batch B | Remaining prod | Sun 22:00–02:00 | CAB | Cycle closed |

### 4.2 Rules

1. **Never patch everything at once.** `serial` + waves.
2. **Pre-flight gates:** disk space, backup freshness, host reachable, no existing failed units.
3. **Reboot only if required** (`/var/run/reboot-required`) and only inside the window.
4. **Health check after every host.** A failed check stops the wave (`any_errors_fatal: true`).
5. **Report before and after:** list of packages upgraded, kernel version, reboot taken y/n.
6. Drift detection runs **weekly in `--check` mode** — it must never change anything.

### 4.3 Abort criteria
Stop the cycle and raise an incident if: a host fails its post-patch health check, a host does
not return after reboot within 10 minutes, or more than one host in a wave fails.

### 4.4 Rollback
OS patch rollback is not reliable in-place. Mitigation strategy instead:
* VM snapshot taken before the wave (`gcloud compute disks snapshot`),
* restore from snapshot if a patch breaks the host,
* hold a specific package (`apt-mark hold`) if it is the identified culprit.

---

## 5. Hands-on implementation

### Step 1 — Project layout

```bash
mkdir -p ~/ansible-patching/{roles,reports,templates,inventories}
cd ~/ansible-patching && git init -b main
```

`inventories/estate.ini`:

```ini
[canary]
dev1 ansible_host=10.128.0.31

[nonprod]
dev1 ansible_host=10.128.0.31
test1 ansible_host=10.128.0.32

[prod_a]
app1 ansible_host=10.128.0.11
app2 ansible_host=10.128.0.12

[prod_b]
app3 ansible_host=10.128.0.13
app4 ansible_host=10.128.0.14

[prod:children]
prod_a
prod_b

[all:vars]
patch_window_start=22
patch_window_end=2
min_free_disk_pct=20
health_endpoint=http://localhost:8080/health
```

### Step 2 — Pre-flight check playbook (run it days before)

`preflight.yml`:

```yaml
---
- name: Patch cycle pre-flight checks
  hosts: "{{ target | default('nonprod') }}"
  gather_facts: true
  become: true
  any_errors_fatal: false

  tasks:
    - name: Collect disk usage
      ansible.builtin.shell: df --output=pcent / | tail -1 | tr -dc '0-9'
      register: disk_used
      changed_when: false

    - name: Count pending upgrades
      ansible.builtin.shell: apt-get -s upgrade | grep -c '^Inst' || true
      register: pending
      changed_when: false

    - name: Detect an already-pending reboot
      ansible.builtin.stat:
        path: /var/run/reboot-required
      register: reboot_pending

    - name: Look for failed systemd units
      ansible.builtin.shell: systemctl --failed --no-legend | wc -l
      register: failed_units
      changed_when: false

    - name: Record findings
      ansible.builtin.set_fact:
        preflight:
          host: "{{ inventory_hostname }}"
          os: "{{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }}"
          kernel: "{{ ansible_facts['kernel'] }}"
          disk_used_pct: "{{ disk_used.stdout | int }}"
          pending_updates: "{{ pending.stdout | int }}"
          reboot_pending: "{{ reboot_pending.stat.exists }}"
          failed_units: "{{ failed_units.stdout | int }}"
          uptime_days: "{{ (ansible_facts['uptime_seconds'] | int / 86400) | round(1) }}"

    - name: Show findings
      ansible.builtin.debug:
        var: preflight

    - name: Gate - enough free disk
      ansible.builtin.assert:
        that: (100 - (disk_used.stdout | int)) >= (min_free_disk_pct | int)
        fail_msg: "{{ inventory_hostname }}: only {{ 100 - (disk_used.stdout|int) }}% free, need {{ min_free_disk_pct }}%"
        success_msg: "disk OK"

    - name: Gate - no pre-existing failed units
      ansible.builtin.assert:
        that: failed_units.stdout | int == 0
        fail_msg: "{{ inventory_hostname }} has {{ failed_units.stdout }} failed unit(s) BEFORE patching - fix first"

- name: Build the pre-flight report
  hosts: localhost
  gather_facts: false
  tasks:
    - name: Render CSV
      ansible.builtin.template:
        src: templates/preflight.csv.j2
        dest: "reports/preflight-{{ lookup('pipe', 'date +%Y%m%d-%H%M') }}.csv"
      delegate_to: localhost
```

`templates/preflight.csv.j2`:

```jinja
host,os,kernel,disk_used_pct,pending_updates,reboot_pending,failed_units,uptime_days
{% for h in groups[target | default('nonprod')] %}
{{ hostvars[h].preflight.host }},{{ hostvars[h].preflight.os }},{{ hostvars[h].preflight.kernel }},{{ hostvars[h].preflight.disk_used_pct }},{{ hostvars[h].preflight.pending_updates }},{{ hostvars[h].preflight.reboot_pending }},{{ hostvars[h].preflight.failed_units }},{{ hostvars[h].preflight.uptime_days }}
{% endfor %}
```

### Step 3 — The patching playbook

`patch.yml`:

```yaml
---
- name: Monthly OS patch cycle
  hosts: "{{ target | mandatory }}"
  serial: "{{ batch | default(1) }}"
  any_errors_fatal: true          # one failure stops the whole wave
  become: true
  gather_facts: true

  vars:
    allow_reboot: "{{ reboot_allowed | default(false) | bool }}"

  pre_tasks:
    - name: Record the pre-patch state
      ansible.builtin.set_fact:
        pre_kernel: "{{ ansible_facts['kernel'] }}"

    - name: List packages that will be upgraded (evidence)
      ansible.builtin.shell: apt-get -s upgrade | awk '/^Inst/ {print $2" "$3}'
      register: to_upgrade
      changed_when: false

    - name: Show the planned upgrade list
      ansible.builtin.debug:
        var: to_upgrade.stdout_lines

    - name: Abort if there is nothing to do
      ansible.builtin.meta: end_host
      when: to_upgrade.stdout_lines | length == 0

  tasks:
    - name: Update the package index
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 0

    - name: Apply upgrades
      ansible.builtin.apt:
        upgrade: dist
        autoremove: true
        autoclean: true
      register: upgrade_result

    - name: Check whether a reboot is required
      ansible.builtin.stat:
        path: /var/run/reboot-required
      register: needs_reboot

    - name: Check we are inside the approved reboot window
      ansible.builtin.set_fact:
        in_window: >-
          {{ (ansible_facts['date_time']['hour'] | int >= patch_window_start | int)
             or (ansible_facts['date_time']['hour'] | int < patch_window_end | int) }}

    - name: Reboot (only if required, allowed and in-window)
      ansible.builtin.reboot:
        msg: "Monthly patch cycle reboot by Ansible"
        reboot_timeout: 600
        connect_timeout: 20
        post_reboot_delay: 30
        test_command: systemctl is-system-running --wait
      when:
        - needs_reboot.stat.exists
        - allow_reboot | bool
        - in_window | bool

    - name: Warn if a reboot is required but not permitted now
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }}: REBOOT PENDING - schedule in the next window"
      when:
        - needs_reboot.stat.exists
        - not (allow_reboot | bool and in_window | bool)

    - name: Wait for the host to be reachable again
      ansible.builtin.wait_for_connection:
        delay: 10
        timeout: 600

    - name: Re-gather facts after reboot
      ansible.builtin.setup:

  post_tasks:
    - name: Post-check - no failed units
      ansible.builtin.shell: systemctl --failed --no-legend | wc -l
      register: post_failed
      changed_when: false

    - name: Gate - services healthy
      ansible.builtin.assert:
        that: post_failed.stdout | int == 0
        fail_msg: "{{ inventory_hostname }}: {{ post_failed.stdout }} failed unit(s) AFTER patching - WAVE ABORTED"

    - name: Post-check - application health endpoint
      ansible.builtin.uri:
        url: "{{ health_endpoint }}"
        status_code: 200
      register: app_health
      retries: 10
      delay: 6
      until: app_health.status == 200
      failed_when: app_health.status != 200
      when: inventory_hostname in groups.get('prod', []) or inventory_hostname in groups.get('nonprod', [])
      ignore_errors: false

    - name: Summarise the result for this host
      ansible.builtin.set_fact:
        patch_result:
          host: "{{ inventory_hostname }}"
          packages_upgraded: "{{ to_upgrade.stdout_lines | length }}"
          kernel_before: "{{ pre_kernel }}"
          kernel_after: "{{ ansible_facts['kernel'] }}"
          rebooted: "{{ needs_reboot.stat.exists and allow_reboot | bool }}"
          failed_units_after: "{{ post_failed.stdout | int }}"
          status: "SUCCESS"

    - name: Print the summary
      ansible.builtin.debug:
        var: patch_result
```

### Step 4 — Run the waves

```bash
# Wave 0 - canary, with reboot
ansible-playbook -i inventories/estate.ini patch.yml \
  -e "target=canary reboot_allowed=true" --diff

# Preview on non-prod WITHOUT changing anything
ansible-playbook -i inventories/estate.ini patch.yml \
  -e "target=nonprod" --check --diff

# Wave 1 - non-prod, 2 hosts at a time
ansible-playbook -i inventories/estate.ini patch.yml \
  -e "target=nonprod batch=2 reboot_allowed=true"

# Wave 2 - prod batch A, strictly one at a time
ansible-playbook -i inventories/estate.ini patch.yml \
  -e "target=prod_a batch=1 reboot_allowed=true"

# Wave 3
ansible-playbook -i inventories/estate.ini patch.yml \
  -e "target=prod_b batch=1 reboot_allowed=true"
```

### Step 5 — Drift detection (read-only compliance run)

`drift-check.yml`:

```yaml
---
# Runs weekly in --check mode. It must NEVER change a managed node.
- name: Configuration drift detection
  hosts: all
  gather_facts: true
  become: true
  check_mode: true          # forced read-only, even without --check

  vars:
    required_packages: [vim, curl, git, chrony, ufw]
    forbidden_packages: [telnet, rsh-client]
    required_services: [ssh, chrony]
    expected_timezone: "Asia/Kolkata"

  tasks:
    - name: Required packages present?
      ansible.builtin.apt:
        name: "{{ required_packages }}"
        state: present
      register: pkg_drift

    - name: Forbidden packages absent?
      ansible.builtin.apt:
        name: "{{ forbidden_packages }}"
        state: absent
      register: forbidden_drift

    - name: Required services running and enabled?
      ansible.builtin.systemd:
        name: "{{ item }}"
        state: started
        enabled: true
      loop: "{{ required_services }}"
      register: svc_drift

    - name: Timezone correct?
      ansible.builtin.command: timedatectl show -p Timezone --value
      register: tz
      changed_when: false

    - name: Root SSH login disabled?
      ansible.builtin.shell: >
        sshd -T 2>/dev/null | awk '/^permitrootlogin/ {print $2}'
      register: root_login
      changed_when: false

    - name: Pending security updates?
      ansible.builtin.shell: apt-get -s upgrade | grep -c '^Inst.*security' || true
      register: sec_updates
      changed_when: false

    - name: Build the compliance record
      ansible.builtin.set_fact:
        compliance:
          host: "{{ inventory_hostname }}"
          os: "{{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }}"
          kernel: "{{ ansible_facts['kernel'] }}"
          package_drift: "{{ pkg_drift.changed }}"
          forbidden_present: "{{ forbidden_drift.changed }}"
          service_drift: "{{ svc_drift.changed }}"
          timezone_ok: "{{ tz.stdout == expected_timezone }}"
          root_login_disabled: "{{ root_login.stdout in ['no', 'prohibit-password'] }}"
          pending_security_updates: "{{ sec_updates.stdout | int }}"
          compliant: >-
            {{ (not pkg_drift.changed)
               and (not forbidden_drift.changed)
               and (not svc_drift.changed)
               and (tz.stdout == expected_timezone)
               and (root_login.stdout in ['no','prohibit-password'])
               and (sec_updates.stdout | int == 0) }}

    - name: Report
      ansible.builtin.debug:
        var: compliance

- name: Render the compliance report
  hosts: localhost
  gather_facts: false
  tasks:
    - name: Write HTML
      ansible.builtin.template:
        src: templates/compliance.html.j2
        dest: "reports/compliance-{{ lookup('pipe', 'date +%Y%m%d') }}.html"
```

`templates/compliance.html.j2`:

```jinja
<!doctype html>
<html><head><meta charset="utf-8"><title>Estate Compliance Report</title>
<style>
 body{font-family:system-ui,sans-serif;margin:2rem}
 table{border-collapse:collapse;width:100%}
 th,td{border:1px solid #ccc;padding:6px 10px;font-size:14px;text-align:left}
 th{background:#f2f2f2}
 .ok{background:#e6ffed}.bad{background:#ffe6e6;font-weight:600}
</style></head><body>
<h1>Estate Compliance Report</h1>
<p>Generated: {{ lookup('pipe','date -Is') }} &middot; Hosts: {{ groups['all'] | length }}</p>
<table>
<tr><th>Host</th><th>OS</th><th>Kernel</th><th>Pkg drift</th><th>Forbidden pkgs</th>
    <th>Service drift</th><th>Timezone</th><th>Root SSH disabled</th>
    <th>Pending security updates</th><th>Result</th></tr>
{% for h in groups['all'] %}
{% set c = hostvars[h].compliance %}
<tr class="{{ 'ok' if c.compliant else 'bad' }}">
  <td>{{ c.host }}</td><td>{{ c.os }}</td><td>{{ c.kernel }}</td>
  <td>{{ 'DRIFT' if c.package_drift else 'OK' }}</td>
  <td>{{ 'FOUND' if c.forbidden_present else 'OK' }}</td>
  <td>{{ 'DRIFT' if c.service_drift else 'OK' }}</td>
  <td>{{ 'OK' if c.timezone_ok else 'WRONG' }}</td>
  <td>{{ 'OK' if c.root_login_disabled else 'ENABLED' }}</td>
  <td>{{ c.pending_security_updates }}</td>
  <td>{{ 'COMPLIANT' if c.compliant else 'NON-COMPLIANT' }}</td>
</tr>
{% endfor %}
</table>
</body></html>
```

```bash
ansible-playbook -i inventories/estate.ini drift-check.yml
xdg-open reports/compliance-$(date +%Y%m%d).html 2>/dev/null || \
  echo "open reports/compliance-$(date +%Y%m%d).html"
```

### Step 6 — Snapshot safety net before a prod wave

```bash
for h in app1 app2; do
  gcloud compute disks snapshot "$h" \
    --snapshot-names "prepatch-${h}-$(date +%Y%m%d)" --zone <zone>
done
gcloud compute snapshots list --filter="name~prepatch"
```

### Step 7 — Prove the abort behaviour

```bash
# Break one host deliberately
ansible app2 -i inventories/estate.ini -m ansible.builtin.systemd \
  -a "name=tomcat state=stopped" --become

# The wave must abort at app2 and leave app3/app4 untouched
ansible-playbook -i inventories/estate.ini patch.yml -e "target=prod batch=1"
# Expect: fatal on app2, PLAY RECAP shows later hosts skipped
```

### Step 8 — Schedule it

```bash
# Weekly drift detection (read-only)
( crontab -l 2>/dev/null; \
  echo "0 6 * * 1 cd $HOME/ansible-patching && ansible-playbook -i inventories/estate.ini drift-check.yml >> /var/tmp/drift.log 2>&1" ) | crontab -
crontab -l
```

---

## 6. Acceptance criteria

- [ ] Pre-flight playbook gates on disk, failed units and reachability, and produces a CSV
- [ ] Patch playbook upgrades, reboots **only when required**, and waits for the host to return
- [ ] `any_errors_fatal: true` proven to abort the wave on a failed host
- [ ] Post-patch health gate uses a real HTTP check
- [ ] Drift detection runs in forced `check_mode` and changes nothing (verify with `--diff`)
- [ ] HTML compliance report generated with a per-host COMPLIANT/NON-COMPLIANT verdict
- [ ] Snapshots taken before the production wave
- [ ] Evidence of a kernel version change before/after

---

## 7. Troubleshooting

| Symptom | Cause | Action |
|---|---|---|
| Host never returns after reboot | Boot failure or firewall | Console/serial log in GCP; restore from the pre-patch snapshot |
| `reboot` task times out | `reboot_timeout` too short for slow boot | Raise `reboot_timeout`, use `test_command` |
| `apt` fails: "Could not get lock" | `unattended-upgrades` running | Wait, or stop the timer during the window |
| Drift check reports changes every week | `check_mode` comparing a non-idempotent task | Replace `shell` with a module, or set `changed_when: false` |
| Whole estate patched at once | `serial` missing or set too high | `serial: 1` for prod waves |
| `end_host` doesn't skip the host | Used in `tasks` instead of `pre_tasks` | `meta: end_host` must run before the work |
| Health check 404 | Endpoint path changed | Parameterise `health_endpoint` per group |
| Report template fails | A host has no `compliance` fact (unreachable) | Use `hostvars[h].compliance | default({})` and show UNREACHABLE |

---

## 8. Cleanup

```bash
crontab -l | grep -v drift-check | crontab -
gcloud compute snapshots delete prepatch-app1-$(date +%Y%m%d) --quiet
rm -rf reports/*
```

---

## 9. Deliverables

1. Repo `ansible-patching`: `preflight.yml`, `patch.yml`, `drift-check.yml`, templates, inventory.
2. `PATCH-SOP-012.md` — waves table, gates, abort criteria, snapshot/rollback strategy.
3. A real pre-flight CSV and a real HTML compliance report.
4. Evidence of an aborted wave (`any_errors_fatal`) with later hosts untouched.
5. Kernel-version before/after table for the patched hosts.

---

## 10. Interview talking points

* *"How do you patch a large estate safely?"* — Waves (canary → non-prod → prod batches),
  `serial`, pre-flight gates, health checks after each host, abort on first failure.
* *"How do you avoid unexpected reboots?"* — Reboot only when `/var/run/reboot-required` exists,
  only when explicitly allowed, and only inside the approved window.
* *"How do you prove compliance to an auditor?"* — A scheduled read-only `check_mode` run that
  renders a dated per-host report; `changed=true` in check mode *is* the drift signal.
* *"What's your rollback plan for OS patches?"* — Pre-wave disk snapshots, restore if needed,
  `apt-mark hold` for an identified bad package. In-place patch rollback is not dependable.
* *"Why `any_errors_fatal`?"* — If one host breaks after patching, the same patch will probably
  break the next one; stop and investigate rather than damaging the whole tier.

**CV bullet:** *Automated the monthly Linux patch cycle and weekly configuration-drift audit with
Ansible — pre-flight gating, wave-based rollout with health checks and automatic abort,
conditional reboots and generated HTML compliance reporting — replacing a four-engineer manual
weekend process.*
