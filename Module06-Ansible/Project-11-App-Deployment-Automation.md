# Project 11 — Zero-Touch Tomcat Application Deployment with Ansible

| | |
|---|---|
| **Module** | 6 — Ansible |
| **Level** | L2 — Engineer |
| **Role you play** | Automation Engineer in a release/deployment team |
| **Scope** | `Ansible-hands-on.md` — inventory, ad-hoc, playbooks, roles, variables, handlers, templates |

---

## 1. Business problem

Deploying the Java application to four Tomcat servers is a 90-minute manual ritual: SSH to each
box, stop Tomcat, back up the old WAR, copy the new one, edit the properties file, start Tomcat,
curl the health endpoint. It is error-prone, it happens after hours, and the runbook is a Word
document that is always out of date.

Deliver a **one-command, idempotent deployment** with Ansible that works on 1 server or 100, with
pre-checks, backup, rolling deployment and automatic verification.

## 2. Objectives

1. Build an inventory with groups and group/host variables.
2. Write a role that installs Java + Tomcat, templates the config, and manages the service.
3. Write a deployment playbook with **backup → deploy → verify → rollback-on-failure**.
4. Deploy **one server at a time** (`serial`) so the service stays available.
5. Prove idempotence: second run changes nothing.

## 3. Scope

**In scope:** inventory (INI/YAML), `ansible` ad-hoc, playbooks, `ansible.builtin.*` FQCN
modules (`apt`, `copy`, `template`, `service`, `systemd`, `user`, `file`, `unarchive`, `uri`,
`stat`, `get_url`, `lineinfile`, `debug`, `assert`, `fail`, `wait_for`), variables, facts,
handlers, loops, conditionals, `when`, tags, roles, `serial`, `block/rescue/always`, Ansible Vault.

**Out of scope:** AWX/Tower, dynamic inventory, custom modules, CI integration (Module 8).

---

## 4. Standard Operating Procedure (SOP)

> **SOP ID:** REL-SOP-011 · **Title:** Application deployment via Ansible

### 4.1 Rules

1. **The playbook is the runbook.** No step exists outside version control.
2. **Idempotent by default.** Re-running a deployment must be safe.
3. **`--check --diff` first** on production. It is the change preview.
4. **`serial: 1`** for stateless app servers so the pool never drops below capacity.
5. **Secrets in Ansible Vault**, never in plain `group_vars`.
6. **Always back up the artefact being replaced**, with a timestamp.
7. **Verify with an HTTP health check, not "the service started"**.
8. `block/rescue` restores the previous WAR automatically if verification fails.

### 4.2 Deployment procedure

| # | Step | Gate |
|---|---|---|
| 1 | `ansible all -m ping` | All hosts reachable |
| 2 | `ansible-playbook deploy.yml --check --diff --limit staging` | Review predicted changes |
| 3 | Deploy to staging | Health check passes |
| 4 | Smoke test staging | Manual/automated |
| 5 | `--check` against production | Reviewed by a second engineer |
| 6 | Deploy production with `serial: 1` | Health check per host |
| 7 | Post-deploy verification across the pool | All 200 OK |
| 8 | Record the deployed version | Release ticket |

### 4.3 Rollback
`ansible-playbook rollback.yml -e "rollback_version=<previous>"` restores the timestamped backup
WAR and restarts Tomcat, host by host.

---

## 5. Hands-on implementation

### Step 1 — Install Ansible on the control node (Ubuntu 26.04)

```bash
# Option A - distro package
sudo apt-get update && sudo apt-get install -y ansible

# Option B - PPA for a newer release
# sudo apt-get install -y software-properties-common
# sudo add-apt-repository --yes --update ppa:ansible/ansible
# sudo apt-get install -y ansible

# Option C - pipx (PEP 668 safe; the system Python is externally managed on 26.04)
# sudo apt-get install -y pipx && pipx install --include-deps ansible && pipx ensurepath

ansible --version
```

### Step 2 — SSH key distribution to the managed nodes

```bash
ssh-keygen -t ed25519 -C "ansible-control" -f ~/.ssh/ansible_ed25519 -N ""
for h in app1 app2 app3 app4; do
  ssh-copy-id -i ~/.ssh/ansible_ed25519.pub ubuntu@$h
done
```

### Step 3 — Project layout

```bash
mkdir -p ~/ansible-deploy/{inventories/{staging,production},roles,files,group_vars,host_vars}
cd ~/ansible-deploy
git init -b main
```

`ansible.cfg`:

```ini
[defaults]
inventory            = inventories/staging/hosts.ini
remote_user          = ubuntu
private_key_file     = ~/.ssh/ansible_ed25519
host_key_checking    = False
retry_files_enabled  = False
stdout_callback      = yaml
roles_path           = roles
forks                = 10

[privilege_escalation]
become        = True
become_method = sudo
become_user   = root
become_ask_pass = False
```

`inventories/production/hosts.ini`:

```ini
[appservers]
app1 ansible_host=10.128.0.11
app2 ansible_host=10.128.0.12
app3 ansible_host=10.128.0.13
app4 ansible_host=10.128.0.14

[appservers:vars]
tomcat_port=8080
app_env=production

[webservers]
web1 ansible_host=10.128.0.21

[production:children]
appservers
webservers
```

`group_vars/appservers.yml`:

```yaml
---
java_package: openjdk-21-jdk
tomcat_version: "10.1.34"
tomcat_user: tomcat
tomcat_group: tomcat
tomcat_home: /opt/tomcat
tomcat_port: 8080
tomcat_shutdown_port: 8005
app_name: orders
app_war_source: "files/{{ app_name }}-{{ app_version }}.war"
app_version: "1.4.0"
backup_dir: /opt/backups/wars
health_url: "http://localhost:{{ tomcat_port }}/{{ app_name }}/health"
health_retries: 15
health_delay: 4
```

### Step 4 — Ad-hoc commands (the daily driver)

```bash
ansible all -i inventories/production/hosts.ini -m ansible.builtin.ping
ansible appservers -m ansible.builtin.setup -a "filter=ansible_distribution*"
ansible appservers -m ansible.builtin.command -a "uptime"
ansible appservers -m ansible.builtin.shell -a "df -h / | tail -1"
ansible appservers -m ansible.builtin.service -a "name=tomcat state=started" --check
ansible appservers -m ansible.builtin.apt -a "update_cache=yes" --become
```

### Step 5 — The `tomcat` role

```bash
mkdir -p roles/tomcat/{tasks,handlers,templates,defaults,files,meta}
```

`roles/tomcat/defaults/main.yml`:

```yaml
---
java_package: openjdk-21-jdk
tomcat_version: "10.1.34"
tomcat_user: tomcat
tomcat_group: tomcat
tomcat_home: /opt/tomcat
tomcat_port: 8080
tomcat_shutdown_port: 8005
tomcat_max_threads: 200
tomcat_xms: 512m
tomcat_xmx: 1024m
```

`roles/tomcat/tasks/main.yml`:

```yaml
---
- name: Fail fast on an unsupported platform
  ansible.builtin.assert:
    that:
      - ansible_facts['os_family'] == 'Debian'
      - ansible_facts['distribution_major_version'] | int >= 24
    fail_msg: "This role supports Ubuntu 24.04+ only (found {{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }})"

- name: Install Java (Debian family)
  ansible.builtin.apt:
    name: "{{ java_package }}"
    state: present
    update_cache: true
    cache_valid_time: 3600
  when: ansible_facts['os_family'] == 'Debian'

- name: Install Java (RedHat family)
  ansible.builtin.dnf:
    name: java-21-openjdk-devel
    state: present
  when: ansible_facts['os_family'] == 'RedHat'

- name: Create the tomcat group
  ansible.builtin.group:
    name: "{{ tomcat_group }}"
    system: true
    state: present

- name: Create the tomcat service account
  ansible.builtin.user:
    name: "{{ tomcat_user }}"
    group: "{{ tomcat_group }}"
    home: "{{ tomcat_home }}"
    shell: /usr/sbin/nologin
    system: true
    create_home: false

- name: Create the tomcat directory
  ansible.builtin.file:
    path: "{{ tomcat_home }}"
    state: directory
    owner: "{{ tomcat_user }}"
    group: "{{ tomcat_group }}"
    mode: "0750"

- name: Download Tomcat
  ansible.builtin.get_url:
    url: "https://dlcdn.apache.org/tomcat/tomcat-10/v{{ tomcat_version }}/bin/apache-tomcat-{{ tomcat_version }}.tar.gz"
    dest: "/tmp/apache-tomcat-{{ tomcat_version }}.tar.gz"
    mode: "0644"

- name: Unpack Tomcat (idempotent via creates)
  ansible.builtin.unarchive:
    src: "/tmp/apache-tomcat-{{ tomcat_version }}.tar.gz"
    dest: "{{ tomcat_home }}"
    remote_src: true
    extra_opts: ['--strip-components=1']
    owner: "{{ tomcat_user }}"
    group: "{{ tomcat_group }}"
    creates: "{{ tomcat_home }}/bin/catalina.sh"

- name: Ensure scripts are executable
  ansible.builtin.file:
    path: "{{ tomcat_home }}/bin"
    state: directory
    recurse: true
    mode: "u+x"
    owner: "{{ tomcat_user }}"
    group: "{{ tomcat_group }}"

- name: Template server.xml
  ansible.builtin.template:
    src: server.xml.j2
    dest: "{{ tomcat_home }}/conf/server.xml"
    owner: "{{ tomcat_user }}"
    group: "{{ tomcat_group }}"
    mode: "0640"
    backup: true
  notify: Restart tomcat

- name: Template the systemd unit
  ansible.builtin.template:
    src: tomcat.service.j2
    dest: /etc/systemd/system/tomcat.service
    owner: root
    group: root
    mode: "0644"
  notify:
    - Reload systemd
    - Restart tomcat

- name: Enable and start tomcat
  ansible.builtin.systemd:
    name: tomcat
    state: started
    enabled: true
    daemon_reload: true
```

`roles/tomcat/handlers/main.yml`:

```yaml
---
- name: Reload systemd
  ansible.builtin.systemd:
    daemon_reload: true

- name: Restart tomcat
  ansible.builtin.systemd:
    name: tomcat
    state: restarted
```

`roles/tomcat/templates/tomcat.service.j2`:

```jinja
# MANAGED BY ANSIBLE - do not edit on the server
[Unit]
Description=Apache Tomcat {{ tomcat_version }} ({{ app_env | default('unset') }})
After=network.target

[Service]
Type=forking
User={{ tomcat_user }}
Group={{ tomcat_group }}
Environment="JAVA_HOME=/usr/lib/jvm/java-21-openjdk-{{ ansible_facts['architecture'] | regex_replace('x86_64','amd64') }}"
Environment="CATALINA_HOME={{ tomcat_home }}"
Environment="CATALINA_BASE={{ tomcat_home }}"
Environment="CATALINA_PID={{ tomcat_home }}/temp/tomcat.pid"
Environment="CATALINA_OPTS=-Xms{{ tomcat_xms }} -Xmx{{ tomcat_xmx }} -server"
ExecStart={{ tomcat_home }}/bin/startup.sh
ExecStop={{ tomcat_home }}/bin/shutdown.sh
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

`roles/tomcat/templates/server.xml.j2` (trimmed to what we manage):

```jinja
<?xml version="1.0" encoding="UTF-8"?>
<!-- MANAGED BY ANSIBLE for {{ inventory_hostname }} ({{ app_env | default('dev') }}) -->
<Server port="{{ tomcat_shutdown_port }}" shutdown="SHUTDOWN">
  <Service name="Catalina">
    <Connector port="{{ tomcat_port }}"
               protocol="HTTP/1.1"
               maxThreads="{{ tomcat_max_threads }}"
               connectionTimeout="20000"
               redirectPort="8443"
               server="ACME" />
    <Engine name="Catalina" defaultHost="localhost">
      <Host name="localhost" appBase="webapps" unpackWARs="true" autoDeploy="true">
        <Valve className="org.apache.catalina.valves.AccessLogValve"
               directory="logs" prefix="access" suffix=".log"
               pattern="%h %l %u %t &quot;%r&quot; %s %b %D" />
      </Host>
    </Engine>
  </Service>
</Server>
```

### Step 6 — The deployment playbook

`deploy.yml`:

```yaml
---
- name: Deploy {{ app_name | default('orders') }} to Tomcat servers
  hosts: appservers
  serial: 1                 # one host at a time - the pool stays up
  max_fail_percentage: 0    # stop the whole rollout on the first failure
  become: true

  vars:
    deploy_stamp: "{{ ansible_date_time.iso8601_basic_short }}"

  pre_tasks:
    - name: Pre-check - disk space
      ansible.builtin.shell: df --output=pcent / | tail -1 | tr -dc '0-9'
      register: disk_pct
      changed_when: false

    - name: Fail if the disk is nearly full
      ansible.builtin.assert:
        that: disk_pct.stdout | int < 85
        fail_msg: "Root filesystem at {{ disk_pct.stdout }}% - deployment aborted"

    - name: Pre-check - the WAR exists on the control node
      ansible.builtin.stat:
        path: "{{ app_war_source }}"
      delegate_to: localhost
      become: false
      register: war_local

    - name: Fail if the artefact is missing
      ansible.builtin.fail:
        msg: "Artefact {{ app_war_source }} not found on the control node"
      when: not war_local.stat.exists

  roles:
    - role: tomcat
      tags: [provision]

  tasks:
    - name: Ensure the backup directory exists
      ansible.builtin.file:
        path: "{{ backup_dir }}"
        state: directory
        mode: "0750"
      tags: [deploy]

    - name: Check for an existing deployment
      ansible.builtin.stat:
        path: "{{ tomcat_home }}/webapps/{{ app_name }}.war"
      register: current_war
      tags: [deploy]

    - name: Back up the current WAR
      ansible.builtin.copy:
        src: "{{ tomcat_home }}/webapps/{{ app_name }}.war"
        dest: "{{ backup_dir }}/{{ app_name }}-{{ deploy_stamp }}.war"
        remote_src: true
        mode: "0640"
      when: current_war.stat.exists
      tags: [deploy]

    - name: Deploy with automatic rollback on failure
      block:
        - name: Stop tomcat
          ansible.builtin.systemd:
            name: tomcat
            state: stopped

        - name: Remove the exploded webapp directory
          ansible.builtin.file:
            path: "{{ tomcat_home }}/webapps/{{ app_name }}"
            state: absent

        - name: Copy the new WAR
          ansible.builtin.copy:
            src: "{{ app_war_source }}"
            dest: "{{ tomcat_home }}/webapps/{{ app_name }}.war"
            owner: "{{ tomcat_user }}"
            group: "{{ tomcat_group }}"
            mode: "0640"

        - name: Template the application properties
          ansible.builtin.template:
            src: application.properties.j2
            dest: "{{ tomcat_home }}/conf/application.properties"
            owner: "{{ tomcat_user }}"
            mode: "0640"

        - name: Start tomcat
          ansible.builtin.systemd:
            name: tomcat
            state: started

        - name: Wait for the port to accept connections
          ansible.builtin.wait_for:
            port: "{{ tomcat_port }}"
            host: 127.0.0.1
            delay: 5
            timeout: 120

        - name: HTTP health check (the real gate)
          ansible.builtin.uri:
            url: "{{ health_url }}"
            status_code: 200
            return_content: true
          register: health
          retries: "{{ health_retries }}"
          delay: "{{ health_delay }}"
          until: health.status == 200

        - name: Report success
          ansible.builtin.debug:
            msg: "{{ inventory_hostname }}: {{ app_name }} {{ app_version }} healthy"

      rescue:
        - name: ROLLBACK - restore the previous WAR
          ansible.builtin.copy:
            src: "{{ backup_dir }}/{{ app_name }}-{{ deploy_stamp }}.war"
            dest: "{{ tomcat_home }}/webapps/{{ app_name }}.war"
            remote_src: true
            owner: "{{ tomcat_user }}"
            mode: "0640"
          when: current_war.stat.exists

        - name: ROLLBACK - restart tomcat
          ansible.builtin.systemd:
            name: tomcat
            state: restarted

        - name: Abort the rollout
          ansible.builtin.fail:
            msg: "Deployment failed on {{ inventory_hostname }} - rolled back. Rollout stopped."

      always:
        - name: Collect the last log lines for the ticket
          ansible.builtin.shell: tail -50 {{ tomcat_home }}/logs/catalina.out
          register: catalina_tail
          changed_when: false
          failed_when: false

        - name: Show log tail
          ansible.builtin.debug:
            var: catalina_tail.stdout_lines
      tags: [deploy]

  post_tasks:
    - name: Prune backups older than 30 days
      ansible.builtin.find:
        paths: "{{ backup_dir }}"
        age: 30d
        patterns: "*.war"
      register: old_backups

    - name: Delete old backups
      ansible.builtin.file:
        path: "{{ item.path }}"
        state: absent
      loop: "{{ old_backups.files }}"
      loop_control:
        label: "{{ item.path }}"
```

`roles/tomcat/templates/application.properties.j2` (place under the playbook's `templates/`):

```jinja
# MANAGED BY ANSIBLE
app.name={{ app_name }}
app.version={{ app_version }}
app.environment={{ app_env }}
server.port={{ tomcat_port }}
db.url=jdbc:mysql://{{ db_host | default('localhost') }}:3306/{{ app_name }}
db.user={{ db_user | default('appuser') }}
db.password={{ vault_db_password }}
```

### Step 7 — Secrets with Ansible Vault

```bash
ansible-vault create group_vars/appservers/vault.yml
# vault_db_password: <STRONG_PASSWORD>

ansible-vault view group_vars/appservers/vault.yml
ansible-vault edit  group_vars/appservers/vault.yml

ansible-playbook deploy.yml --ask-vault-pass
# or non-interactively in CI:
# ansible-playbook deploy.yml --vault-password-file ~/.vault_pass   (mode 600, gitignored)
```

### Step 8 — Run it

```bash
# Syntax and lint
ansible-playbook deploy.yml --syntax-check
ansible-lint deploy.yml || true

# THE CHANGE PREVIEW
ansible-playbook -i inventories/staging/hosts.ini deploy.yml --check --diff

# Staging
ansible-playbook -i inventories/staging/hosts.ini deploy.yml --ask-vault-pass

# Production, one host at a time
ansible-playbook -i inventories/production/hosts.ini deploy.yml --ask-vault-pass

# Only redeploy the WAR (skip provisioning)
ansible-playbook -i inventories/production/hosts.ini deploy.yml --tags deploy

# Single host
ansible-playbook -i inventories/production/hosts.ini deploy.yml --limit app2
```

### Step 9 — Prove idempotence

```bash
ansible-playbook -i inventories/production/hosts.ini deploy.yml --tags provision
# Second run PLAY RECAP must show changed=0 for the provisioning tasks
```

### Step 10 — Rollback playbook

`rollback.yml`:

```yaml
---
- name: Roll back {{ app_name }} to a previous artefact
  hosts: appservers
  serial: 1
  become: true
  vars_prompt:
    - name: rollback_file
      prompt: "Backup WAR filename to restore (from {{ '{{ backup_dir }}' }})"
      private: false

  tasks:
    - name: Verify the requested backup exists
      ansible.builtin.stat:
        path: "{{ backup_dir }}/{{ rollback_file }}"
      register: bk

    - name: Fail if missing
      ansible.builtin.fail:
        msg: "{{ backup_dir }}/{{ rollback_file }} not found on {{ inventory_hostname }}"
      when: not bk.stat.exists

    - name: Restore
      ansible.builtin.copy:
        src: "{{ backup_dir }}/{{ rollback_file }}"
        dest: "{{ tomcat_home }}/webapps/{{ app_name }}.war"
        remote_src: true
        owner: "{{ tomcat_user }}"
        mode: "0640"

    - name: Restart tomcat
      ansible.builtin.systemd:
        name: tomcat
        state: restarted

    - name: Health check
      ansible.builtin.uri:
        url: "{{ health_url }}"
        status_code: 200
      register: h
      retries: 15
      delay: 4
      until: h.status == 200
```

---

## 6. Acceptance criteria

- [ ] `ansible all -m ping` succeeds for every host
- [ ] Role installs Java + Tomcat and the service starts enabled
- [ ] All modules use FQCN (`ansible.builtin.*`)
- [ ] `--check --diff` produces a readable change preview
- [ ] Deployment runs `serial: 1` and the pool stays available
- [ ] Health check (HTTP 200) is the gate, not "service started"
- [ ] A deliberately broken WAR triggers the `rescue` block and auto-rollback
- [ ] Second provisioning run reports `changed=0`
- [ ] DB password stored in Ansible Vault, not in Git

---

## 7. Troubleshooting

| Symptom | Cause | Action |
|---|---|---|
| `Failed to connect ... Permission denied (publickey)` | Key not distributed / wrong `remote_user` | `ssh-copy-id`, check `ansible.cfg` |
| `sudo: a password is required` | No passwordless sudo | `--ask-become-pass` or configure sudoers |
| Task reports `changed` every run | Using `command`/`shell` without a guard | Use a proper module, or `creates:`/`changed_when:` |
| `unarchive` re-extracts each run | Missing `creates:` | Add `creates: <known file>` |
| `uri` health check fails but the app works | Wrong path or Tomcat still deploying | Increase `retries`/`delay`; verify the context path |
| Vault variable undefined | `vault.yml` not in a loaded `group_vars` path | Put it in `group_vars/<group>/vault.yml` |
| Template renders `AnsibleUndefinedVariable` | Variable not defined for that host | Add to `group_vars`, or use `| default()` |
| Deployment kills all servers at once | Missing `serial` | `serial: 1` (or a percentage) |
| `JAVA_HOME` wrong in the unit file | Architecture-specific path | Derive it from facts, or use `alternatives` |

---

## 8. Cleanup

```bash
ansible appservers -m ansible.builtin.systemd -a "name=tomcat state=stopped enabled=no" --become
ansible appservers -m ansible.builtin.file -a "path=/opt/tomcat state=absent" --become
ansible appservers -m ansible.builtin.file -a "path=/opt/backups/wars state=absent" --become
```

---

## 9. Deliverables

1. Repo `ansible-deploy`: `ansible.cfg`, inventories, `group_vars`, `roles/tomcat`, `deploy.yml`, `rollback.yml`.
2. `DEPLOY-SOP-011.md` — the rules, the 8-step procedure, rollback instructions.
3. Evidence: `--check --diff` output, a successful rollout `PLAY RECAP`, an idempotent second run.
4. A forced-failure run showing `rescue` rolling back automatically.
5. Before/after timing: manual 90 min vs automated run time.

---

## 10. Interview talking points

* *"How do you deploy without downtime using Ansible?"* — `serial: 1` with an HTTP health gate
  per host and `max_fail_percentage: 0` so a bad host stops the rollout.
* *"How do you make a playbook idempotent?"* — Use real modules instead of `shell`; guard any
  unavoidable commands with `creates`, `changed_when`, `when`.
* *"How do you handle a failed deployment automatically?"* — `block/rescue/always`: back up
  first, deploy in the block, restore in `rescue`, always collect logs.
* *"Where do secrets live?"* — Ansible Vault-encrypted `group_vars`, decrypted at runtime with a
  password file that is never committed.
* *"Ansible vs Puppet?"* — Ansible is agentless and push-based, excellent for ordered,
  orchestrated deployments; Puppet is agent/pull-based and excels at continuous state
  enforcement. Many shops use both.
* *"How do you preview a change on production?"* — `--check --diff`, reviewed by a second
  engineer, limited with `--limit` to a canary host first.

**CV bullet:** *Automated a four-server Java/Tomcat release with Ansible — provisioning role,
templated configuration, Vault-managed secrets, serialised rolling deployment with HTTP health
gating and automatic rollback — cutting a 90-minute manual after-hours process to a single
idempotent command.*
