# Project 09 — Enforce a Server Security Baseline with Puppet

| | |
|---|---|
| **Module** | 5 — Puppet |
| **Level** | L2 — Engineer |
| **Role you play** | Configuration Management Engineer supporting an audit remediation programme |
| **Scope** | `1-Puppet-installation.md`, `2-Puppet-manifest.md`, `3-Variables-in-manifests.md`, `4-Loops`, `5-Conditions` |

---

## 1. Business problem

An internal audit found that the Linux estate does not meet the company baseline: inconsistent
packages, missing banners, wrong timezone, unmanaged users, SSH settings that drift back after
every manual fix. Remediation by hand (Project 01) worked for one server — it does not scale to
40, and the servers drift again within weeks.

You must express the baseline as **Puppet code** so that it is applied continuously, drift is
corrected automatically, and you can produce **evidence of compliance** for the auditor.

## 2. Objectives

1. Install Puppet 8 (agent, masterless/apply mode) on Ubuntu 26.04.
2. Codify the Project 01 baseline as idempotent Puppet manifests.
3. Use **variables, facts, conditionals and loops** so one manifest covers all servers.
4. Demonstrate **drift correction**: break the config by hand, re-run, watch Puppet fix it.
5. Produce **compliance evidence** from `--noop` runs.

## 3. Scope

**In scope:** Puppet resource types `package`, `service`, `file`, `user`, `group`, `exec`,
`notify`; facts; variables; `if`/`unless`/`case`/selectors; `each` loops; arrays and hashes;
resource ordering (`require`, `before`, `notify`, `subscribe`); `puppet apply`, `--noop`,
`puppet parser validate`, `puppet-lint`.

**Out of scope:** modules (Project 10), Hiera, PuppetDB, agent/server certificates at scale.

---

## 4. Standard Operating Procedure (SOP)

> **SOP ID:** CFG-SOP-009 · **Title:** Baseline enforcement with Puppet

### 4.1 Principles

1. **Declare desired state, not steps.** Puppet decides what to change.
2. **Idempotence is mandatory.** Run 1 makes changes; run 2 must report **0 changes**. If run 2
   still changes things, the manifest is buggy.
3. **Never make a manual change on a managed server.** Change the code, then apply it.
4. **`--noop` before every real apply** on anything that matters — it is your change preview.
5. **Order explicitly** where it matters (package → config → service).
6. Code lives in Git; applying unreviewed code to a server is a change-process violation.

### 4.2 Change procedure

| # | Step | Command |
|---|---|---|
| 1 | Lint and syntax check | `puppet parser validate manifests/*.pp` |
| 2 | Dry run on a canary host | `sudo puppet apply --noop site.pp` |
| 3 | Review the proposed change list | read the `--noop` output |
| 4 | Apply to canary | `sudo puppet apply site.pp` |
| 5 | Verify idempotence | re-run; expect no changes |
| 6 | Apply to the rest of the fleet | repeat per host/group |
| 7 | Store the run report as evidence | `--logdest` to a file |

### 4.3 Rollback
Puppet rolls back by **reverting the code in Git and re-applying**. Manual un-doing on the server
is forbidden — the next Puppet run would undo your undo.

---

## 5. Hands-on implementation

### Step 1 — Install Puppet 8 on Ubuntu 26.04

```bash
CODENAME=$(lsb_release -cs)
wget https://apt.puppet.com/puppet8-release-${CODENAME}.deb || \
  wget https://apt.puppet.com/puppet8-release-noble.deb      # fallback if 26.04 suite is not published yet

sudo dpkg -i puppet8-release-*.deb
sudo apt-get update
sudo apt-get install -y puppet-agent

echo 'export PATH=$PATH:/opt/puppetlabs/bin' >> ~/.bashrc
export PATH=$PATH:/opt/puppetlabs/bin

puppet --version
puppet facts | head -30            # the data your manifests can branch on
```

### Step 2 — Project layout

```bash
mkdir -p ~/puppet-baseline/{manifests,files,reports} && cd ~/puppet-baseline
git init -b main
```

### Step 3 — Baseline data (variables, arrays, hashes)

`manifests/params.pp`:

```puppet
# Central data for the baseline. Changing a value here changes the whole fleet.
$org_name          = 'ACME Ltd'
$standard_timezone = 'Asia/Kolkata'

# Array -> consumed by a loop
$baseline_packages = ['vim', 'curl', 'wget', 'git', 'htop', 'unzip', 'jq', 'tree']

# Packages that must NOT be present
$forbidden_packages = ['telnet', 'rsh-client', 'nis']

# Hash -> users with their attributes
$baseline_users = {
  'appuser' => { 'uid' => 10001, 'shell' => '/usr/sbin/nologin', 'groups' => ['appgroup'] },
  'opsuser' => { 'uid' => 10002, 'shell' => '/bin/bash',         'groups' => ['appgroup', 'sudo'] },
}

# Directory tree with modes
$app_dirs = {
  '/opt/app'        => '0750',
  '/opt/app/bin'    => '0750',
  '/opt/app/conf'   => '0750',
  '/opt/app/logs'   => '2770',
  '/opt/app/data'   => '0750',
}
```

### Step 4 — The baseline manifest

`manifests/site.pp`:

```puppet
# ACME Linux baseline - applied to every server.
import 'params.pp'   # for the course lab; real code uses modules (Project 10)

### 1. Facts and conditionals -------------------------------------------------
notify { 'host-summary':
  message => "Applying ${org_name} baseline to ${facts['networking']['hostname']} \
(${facts['os']['name']} ${facts['os']['release']['full']}, ${facts['processors']['count']} CPU)",
}

# Only support the OS family we have certified
unless $facts['os']['family'] == 'Debian' {
  fail("Unsupported OS family: ${facts['os']['family']}. Baseline certified on Debian/Ubuntu only.")
}

# Package manager chosen by OS family (shows the pattern for a mixed estate)
$pkg_provider = $facts['os']['family'] ? {
  'Debian' => 'apt',
  'RedHat' => 'dnf',
  default  => undef,
}

### 2. Loop over required packages --------------------------------------------
$baseline_packages.each |String $pkg| {
  package { $pkg:
    ensure => installed,
  }
}

### 3. Loop over forbidden packages -------------------------------------------
$forbidden_packages.each |String $pkg| {
  package { $pkg:
    ensure => absent,
  }
}

### 4. Groups and users from a hash -------------------------------------------
group { 'appgroup':
  ensure => present,
  gid    => 10001,
}

$baseline_users.each |String $username, Hash $attrs| {
  user { $username:
    ensure     => present,
    uid        => $attrs['uid'],
    shell      => $attrs['shell'],
    gid        => 'appgroup',
    groups     => $attrs['groups'],
    managehome => true,
    require    => Group['appgroup'],
  }
}

### 5. Directory tree from a hash ---------------------------------------------
$app_dirs.each |String $path, String $mode| {
  file { $path:
    ensure  => directory,
    owner   => 'appuser',
    group   => 'appgroup',
    mode    => $mode,
    require => User['appuser'],
  }
}

### 6. Timezone - exec guarded so it is idempotent -----------------------------
exec { 'set-timezone':
  command => "/usr/bin/timedatectl set-timezone ${standard_timezone}",
  unless  => "/usr/bin/test \"$(/usr/bin/timedatectl show -p Timezone --value)\" = '${standard_timezone}'",
  path    => ['/usr/bin', '/bin'],
}

### 7. Managed file from a template-like content ------------------------------
file { '/etc/motd':
  ensure  => file,
  owner   => 'root',
  group   => 'root',
  mode    => '0644',
  content => @("MOTD"/L)
    ***************************************************************
    ${org_name} - AUTHORISED ACCESS ONLY
    Host : ${facts['networking']['fqdn']}
    OS   : ${facts['os']['name']} ${facts['os']['release']['full']}
    This system is managed by Puppet. Manual changes WILL be reverted.
    ***************************************************************
    | MOTD
}

### 8. Service management with config -> service relationship ------------------
package { 'chrony':
  ensure => installed,
}

file { '/etc/chrony/chrony.conf':
  ensure  => file,
  owner   => 'root',
  group   => 'root',
  mode    => '0644',
  source  => '/home/ubuntu/puppet-baseline/files/chrony.conf',
  require => Package['chrony'],
  notify  => Service['chrony'],      # config change restarts the service
}

service { 'chrony':
  ensure    => running,
  enable    => true,
  require   => Package['chrony'],
  subscribe => File['/etc/chrony/chrony.conf'],
}

### 9. Conditional: only on small hosts ---------------------------------------
if Integer($facts['memory']['system']['total_bytes']) < 2147483648 {
  notify { 'low-memory-warning':
    message => 'Host has < 2GB RAM - swap tuning profile should be applied.',
  }
}

### 10. Case statement on role -------------------------------------------------
$server_role = $facts['networking']['hostname'] ? {
  /^web/  => 'webserver',
  /^db/   => 'database',
  default => 'generic',
}

case $server_role {
  'webserver': { notify { 'role': message => 'Role: webserver - nginx profile applies' } }
  'database':  { notify { 'role': message => 'Role: database - tuned profile applies' } }
  default:     { notify { 'role': message => 'Role: generic baseline only' } }
}
```

`files/chrony.conf`:

```bash
mkdir -p files
cat > files/chrony.conf <<'EOF'
pool ntp.ubuntu.com iburst maxsources 4
keyfile /etc/chrony/chrony.keys
driftfile /var/lib/chrony/chrony.drift
logdir /var/log/chrony
maxupdateskew 100.0
rtcsync
makestep 1 3
EOF
```

### Step 5 — Validate, dry run, apply

```bash
cd ~/puppet-baseline

# Syntax
puppet parser validate manifests/*.pp

# Style (optional but expected in a real team)
sudo /opt/puppetlabs/puppet/bin/gem install puppet-lint
/opt/puppetlabs/puppet/bin/puppet-lint --no-140chars-check manifests/

# Dry run - THE CHANGE PREVIEW
sudo puppet apply --noop manifests/site.pp | tee reports/noop-$(date +%F-%H%M).log

# Apply
sudo puppet apply manifests/site.pp | tee reports/apply-$(date +%F-%H%M).log
```

### Step 6 — Prove idempotence (the real test)

```bash
sudo puppet apply manifests/site.pp | tee reports/idempotence-check.log
grep -E "Applied catalog|changed" reports/idempotence-check.log
```

Expected on the **second** run: `Notice: Applied catalog in X seconds` with **no**
`(corrective)` change lines. Any repeated change means a non-idempotent resource — usually an
`exec` missing `unless`/`onlyif`/`creates`.

### Step 7 — Drift correction drill

```bash
# Break it by hand, the way a panicking engineer would at 2am
sudo rm -f /etc/motd
sudo userdel -r opsuser
sudo chmod 777 /opt/app/conf
sudo systemctl stop chrony
sudo apt-get install -y telnet

# Evidence of drift
ls -l /etc/motd 2>&1; id opsuser 2>&1; ls -ld /opt/app/conf; systemctl is-active chrony; which telnet
```

```bash
# Puppet shows exactly what drifted
sudo puppet apply --noop manifests/site.pp | grep -E "current_value|would have"

# Puppet repairs it
sudo puppet apply manifests/site.pp | tee reports/drift-repair.log

# Verify
ls -l /etc/motd; id opsuser; ls -ld /opt/app/conf; systemctl is-active chrony
dpkg -l telnet 2>/dev/null | grep -c '^ii' || echo "telnet removed"
```

### Step 8 — Continuous enforcement

```bash
sudo tee /etc/cron.d/puppet-baseline > /dev/null <<'EOF'
# Enforce the baseline every 30 minutes
*/30 * * * * root /opt/puppetlabs/bin/puppet apply /home/ubuntu/puppet-baseline/manifests/site.pp --logdest /var/log/puppet-baseline.log
EOF

sudo systemctl restart cron
sudo tail -20 /var/log/puppet-baseline.log
```

### Step 9 — Compliance evidence for the auditor

```bash
cat > ~/puppet-baseline/compliance-report.sh <<'EOF'
#!/usr/bin/env bash
# A clean --noop run == the host is compliant.
LOG=/tmp/compliance-$(hostname -s)-$(date +%F).log
sudo /opt/puppetlabs/bin/puppet apply --noop \
  /home/ubuntu/puppet-baseline/manifests/site.pp > "$LOG" 2>&1
DRIFT=$(grep -c "current_value" "$LOG")
echo "HOST=$(hostname -f) DATE=$(date -Is) DRIFTED_RESOURCES=${DRIFT}"
[ "$DRIFT" -eq 0 ] && echo "RESULT: COMPLIANT" || { echo "RESULT: NON-COMPLIANT"; grep "current_value" "$LOG"; }
EOF
chmod +x ~/puppet-baseline/compliance-report.sh
~/puppet-baseline/compliance-report.sh
```

---

## 6. Acceptance criteria

- [ ] `puppet parser validate` passes with no errors
- [ ] First apply makes changes; **second apply makes none** (idempotent)
- [ ] Manifest uses variables, an array loop, a hash loop, `if`/`unless`, and a `case`
- [ ] Facts are used (hostname, OS, memory) rather than hard-coded values
- [ ] All five manual drifts are automatically repaired
- [ ] `--noop` output is readable as a change preview
- [ ] Cron enforcement running and logging
- [ ] Compliance report prints `COMPLIANT` on a clean host

---

## 7. Troubleshooting

| Symptom | Cause | Action |
|---|---|---|
| `puppet: command not found` | Binary not on PATH | `export PATH=$PATH:/opt/puppetlabs/bin` |
| Second run still reports changes | `exec` without a guard | Add `unless`/`onlyif`/`creates` |
| `Could not find dependency` | Resource referenced before defined | Add `require`/`before` or fix the title/case |
| Duplicate declaration error | Same resource declared twice | One declaration per resource; use variables/loops |
| File ownership fails | User does not exist yet | `require => User[...]` |
| Service restarts on every run | `notify` from a file whose content changes each run | Make the content deterministic |
| Package install fails | Stale apt cache | `exec { 'apt-update': }` with `before`, or run `apt-get update` first |
| Repo deb not found for 26.04 | Vendor suite not yet published | Use the previous LTS codename fallback shown in Step 1 |

---

## 8. Cleanup

```bash
sudo rm -f /etc/cron.d/puppet-baseline /var/log/puppet-baseline.log
sudo userdel -r appuser 2>/dev/null; sudo userdel -r opsuser 2>/dev/null
sudo rm -rf /opt/app
gcloud compute instances delete <vm-name> --zone <zone>
```

---

## 9. Deliverables

1. Git repo `puppet-baseline` with `manifests/`, `files/`, `compliance-report.sh`.
2. `BASELINE-SPEC.md` — every enforced control and the business/audit reason for it.
3. Run logs: first apply, idempotence check, drift repair.
4. `DRIFT-DRILL.md` — what you broke, what `--noop` predicted, what Puppet repaired.
5. A sample compliance report for the auditor.

---

## 10. Interview talking points

* *"What is idempotence and why does it matter?"* — Applying the same config repeatedly produces
  the same state; it's what makes continuous enforcement safe. I verify it by running twice.
* *"How do you handle configuration drift?"* — Puppet runs on a schedule, detects divergence from
  declared state and corrects it; `--noop` reports drift without changing anything.
* *"Why not just use a shell script?"* — Scripts describe steps and must handle every prior state
  themselves; Puppet describes desired state, is declarative, reports changes and is testable.
* *"How do you make a change safely?"* — Lint → `--noop` on a canary → review the diff → apply →
  verify idempotence → roll to the fleet; rollback is reverting code in Git.
* *"Where do you put data vs logic?"* — Data (packages, users, timezone) in variables/hashes
  (later: Hiera); logic in manifests. That way one code path serves many environments.

**CV bullet:** *Codified the organisation's Linux security baseline in Puppet 8 (packages, users,
permissions, services, banners) with fact-driven conditionals and loops, scheduled enforcement
and `--noop`-based compliance reporting, eliminating recurring configuration drift across the
server estate.*
