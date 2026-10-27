# Project 10 — Reusable Puppet Module for a Web-Server Fleet

| | |
|---|---|
| **Module** | 5 — Puppet |
| **Level** | L2 — Engineer |
| **Role you play** | Configuration Management Engineer building the platform's reusable modules |
| **Scope** | `6-Modules-in-puppet.md` plus all earlier Module 5 hands-on |

---

## 1. Business problem

Three teams each wrote their own 300-line Puppet manifest to install a web server. They disagree
on the document root, none of them manage the firewall, and when a change is needed somebody has
to edit three files. Copy-paste configuration management is as bad as copy-paste code.

Your job: build **one reusable, parameterised Puppet module** — `acme_webserver` — that all three
teams consume, with environment-specific values supplied as parameters, and publish the
versioning/consumption SOP.

## 2. Objectives

1. Build a module with the standard Puppet layout (`manifests/`, `templates/`, `files/`, `examples/`).
2. Expose typed parameters with sensible defaults (`init.pp` + `params.pp`).
3. Split responsibilities into classes: `install`, `config`, `service`, and a defined type for vhosts.
4. Drive configuration from an **ERB template** using facts and parameters.
5. Deploy three different environments (dev/staging/prod) from the **same module**.
6. Document module versioning and consumer upgrade rules.

## 3. Scope

**In scope:** module directory structure, `class` with parameters and data types, `params.pp`
inheritance pattern, class containment and ordering (`contain`, `->`), `define` (defined
resource type), ERB templates, `validate`/data types, `puppet module` layout, `--modulepath`.

**Out of scope:** Hiera, Forge publishing, PuppetDB/exported resources, r10k.

---

## 4. Standard Operating Procedure (SOP)

> **SOP ID:** CFG-SOP-010 · **Title:** Build, version and consume platform Puppet modules

### 4.1 Module design rules

| # | Rule | Why |
|---|---|---|
| 1 | One module = one piece of software | Reusability |
| 2 | **No hard-coded values** in `install/config/service` | Consumers differ |
| 3 | All parameters typed (`String`, `Integer`, `Boolean`, `Array`, `Enum`) | Fails fast with a clear error |
| 4 | Defaults in `params.pp`, chosen by OS family | Portability |
| 5 | Standard class order: `install -> config -> service` | Deterministic |
| 6 | `config` notifies `service` | Config change restarts the service |
| 7 | Public API is `init.pp` only; sub-classes are private | Safe internal refactoring |
| 8 | Ship a working `examples/` manifest | Consumers copy it, don't guess |
| 9 | Semantic versioning; breaking parameter changes = major | Consumer trust |

### 4.2 Consumer upgrade procedure

| # | Step |
|---|---|
| 1 | Read the module CHANGELOG for breaking changes |
| 2 | Pin the module version in your environment |
| 3 | `--noop` against a canary node |
| 4 | Review the predicted diff |
| 5 | Apply to canary, verify the application still serves traffic |
| 6 | Roll to the remaining nodes |

---

## 5. Hands-on implementation

### Step 1 — Create the module skeleton

```bash
export PATH=$PATH:/opt/puppetlabs/bin
mkdir -p ~/puppet-modules && cd ~/puppet-modules
puppet module generate acme-acme_webserver --skip-interview 2>/dev/null || \
  mkdir -p acme_webserver/{manifests,templates,files,examples,spec}

cd acme_webserver
mkdir -p manifests templates files examples
tree -L 2
```

Target layout:

```
acme_webserver/
├── manifests/
│   ├── init.pp        # public API
│   ├── params.pp      # OS-specific defaults
│   ├── install.pp     # private
│   ├── config.pp      # private
│   ├── service.pp     # private
│   └── vhost.pp       # defined type
├── templates/
│   ├── nginx.conf.erb
│   └── vhost.conf.erb
├── files/
│   └── index.html
├── examples/
│   ├── dev.pp
│   ├── staging.pp
│   └── prod.pp
└── metadata.json
```

### Step 2 — `params.pp` — OS-aware defaults

```puppet
# manifests/params.pp
class acme_webserver::params {

  case $facts['os']['family'] {
    'Debian': {
      $package_name = 'nginx'
      $service_name = 'nginx'
      $config_dir   = '/etc/nginx'
      $vhost_dir    = '/etc/nginx/conf.d'
      $docroot      = '/var/www/html'
      $run_user     = 'www-data'
    }
    'RedHat': {
      $package_name = 'nginx'
      $service_name = 'nginx'
      $config_dir   = '/etc/nginx'
      $vhost_dir    = '/etc/nginx/conf.d'
      $docroot      = '/usr/share/nginx/html'
      $run_user     = 'nginx'
    }
    default: {
      fail("acme_webserver does not support ${facts['os']['family']}")
    }
  }

  # Sensible defaults a consumer can override
  $port            = 80
  $worker_processes = 'auto'
  $worker_connections = 1024
  $keepalive_timeout  = 65
  $manage_firewall = false
  $service_ensure  = 'running'
  $service_enable  = true
}
```

### Step 3 — `init.pp` — the public, typed API

```puppet
# manifests/init.pp
#
# @summary Installs and configures the ACME standard web server.
#
# @param port Listening port.
# @param docroot Document root directory.
# @param worker_processes nginx worker_processes value.
# @param worker_connections Connections per worker.
# @param server_tokens Whether to expose the version banner (security).
# @param vhosts Hash of virtual hosts to create.
# @param environment_name Free-form environment label used in the banner page.
#
# @example Minimal
#   include acme_webserver
#
# @example Production
#   class { 'acme_webserver':
#     port               => 80,
#     worker_processes   => 4,
#     worker_connections => 4096,
#     server_tokens      => false,
#     environment_name   => 'prod',
#   }
class acme_webserver (
  Integer[1, 65535]        $port               = $acme_webserver::params::port,
  Stdlib_compat_absolute   $docroot            = $acme_webserver::params::docroot,
  Variant[Integer, Enum['auto']] $worker_processes = $acme_webserver::params::worker_processes,
  Integer[128, 65536]      $worker_connections = $acme_webserver::params::worker_connections,
  Integer[1, 3600]         $keepalive_timeout  = $acme_webserver::params::keepalive_timeout,
  Boolean                  $server_tokens      = false,
  Enum['running','stopped'] $service_ensure    = $acme_webserver::params::service_ensure,
  Boolean                  $service_enable     = $acme_webserver::params::service_enable,
  Boolean                  $manage_firewall    = $acme_webserver::params::manage_firewall,
  String[1]                $environment_name   = 'dev',
  Hash                     $vhosts             = {},
  String[1]                $package_name       = $acme_webserver::params::package_name,
  String[1]                $service_name       = $acme_webserver::params::service_name,
  String[1]                $config_dir         = $acme_webserver::params::config_dir,
  String[1]                $vhost_dir          = $acme_webserver::params::vhost_dir,
  String[1]                $run_user           = $acme_webserver::params::run_user,
) inherits acme_webserver::params {

  # Standard, deterministic ordering
  contain acme_webserver::install
  contain acme_webserver::config
  contain acme_webserver::service

  Class['acme_webserver::install']
  -> Class['acme_webserver::config']
  ~> Class['acme_webserver::service']

  # Create any vhosts the consumer asked for
  $vhosts.each |String $vhost_name, Hash $vhost_params| {
    acme_webserver::vhost { $vhost_name:
      * => $vhost_params,
    }
  }
}
```

> If `Stdlib_compat_absolute` is unavailable (no stdlib installed), use `String[1]` and validate
> with `assert_type(Pattern[/^\//], $docroot)` — the point is that **parameters are typed**.

### Step 4 — Private classes

```puppet
# manifests/install.pp
class acme_webserver::install {
  assert_private()

  package { $acme_webserver::package_name:
    ensure => installed,
  }
}
```

```puppet
# manifests/config.pp
class acme_webserver::config {
  assert_private()

  file { $acme_webserver::docroot:
    ensure => directory,
    owner  => $acme_webserver::run_user,
    group  => $acme_webserver::run_user,
    mode   => '0755',
  }

  file { "${acme_webserver::config_dir}/nginx.conf":
    ensure  => file,
    owner   => 'root',
    group   => 'root',
    mode    => '0644',
    content => template('acme_webserver/nginx.conf.erb'),
  }

  file { $acme_webserver::vhost_dir:
    ensure  => directory,
    owner   => 'root',
    group   => 'root',
    mode    => '0755',
    purge   => true,      # unmanaged vhosts are removed - no surprise config
    recurse => true,
  }

  file { "${acme_webserver::docroot}/index.html":
    ensure  => file,
    owner   => $acme_webserver::run_user,
    mode    => '0644',
    content => template('acme_webserver/index.html.erb'),
    require => File[$acme_webserver::docroot],
  }
}
```

```puppet
# manifests/service.pp
class acme_webserver::service {
  assert_private()

  service { $acme_webserver::service_name:
    ensure     => $acme_webserver::service_ensure,
    enable     => $acme_webserver::service_enable,
    hasrestart => true,
    hasstatus  => true,
  }
}
```

### Step 5 — Defined type for virtual hosts

```puppet
# manifests/vhost.pp
#
# @summary Creates one nginx virtual host.
define acme_webserver::vhost (
  String[1]         $server_name,
  Stdlib_compat_absolute $docroot = "/var/www/${title}",
  Integer[1, 65535] $port         = 80,
  Boolean           $default      = false,
  Array[String]     $index_files  = ['index.html'],
) {

  include acme_webserver

  file { $docroot:
    ensure => directory,
    owner  => $acme_webserver::run_user,
    mode   => '0755',
  }

  file { "${acme_webserver::vhost_dir}/${title}.conf":
    ensure  => file,
    owner   => 'root',
    mode    => '0644',
    content => template('acme_webserver/vhost.conf.erb'),
    notify  => Service[$acme_webserver::service_name],
    require => File[$docroot],
  }
}
```

### Step 6 — ERB templates

`templates/nginx.conf.erb`:

```erb
# MANAGED BY PUPPET - module acme_webserver
# Host        : <%= scope['facts']['networking']['fqdn'] %>
# Environment : <%= @environment_name %>
# Generated   : do not edit by hand, changes are reverted

user <%= @run_user %>;
worker_processes <%= @worker_processes %>;
pid /run/nginx.pid;

events {
    worker_connections <%= @worker_connections %>;
}

http {
    sendfile on;
    tcp_nopush on;
    keepalive_timeout <%= @keepalive_timeout %>;
    server_tokens <%= @server_tokens ? 'on' : 'off' %>;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    access_log /var/log/nginx/access.log;
    error_log  /var/log/nginx/error.log;

    include <%= @vhost_dir %>/*.conf;
}
```

`templates/vhost.conf.erb`:

```erb
# MANAGED BY PUPPET - vhost <%= @title %>
server {
    listen <%= @port %><%= @default ? ' default_server' : '' %>;
    server_name <%= @server_name %>;
    root <%= @docroot %>;
    index <%= @index_files.join(' ') %>;

    location /health {
        access_log off;
        return 200 "ok\n";
    }

    location / {
        try_files $uri $uri/ =404;
    }
}
```

`templates/index.html.erb`:

```erb
<!doctype html>
<html>
<head><title>ACME <%= @environment_name %></title></head>
<body>
  <h1>ACME Web Server</h1>
  <ul>
    <li>Environment: <%= @environment_name %></li>
    <li>Host: <%= scope['facts']['networking']['hostname'] %></li>
    <li>OS: <%= scope['facts']['os']['name'] %> <%= scope['facts']['os']['release']['full'] %></li>
    <li>Managed by Puppet</li>
  </ul>
</body>
</html>
```

### Step 7 — Consumer manifests (same module, three environments)

`examples/dev.pp`:

```puppet
class { 'acme_webserver':
  environment_name   => 'dev',
  port               => 80,
  worker_processes   => 1,
  worker_connections => 256,
  server_tokens      => true,     # dev wants the version visible for debugging
}
```

`examples/prod.pp`:

```puppet
class { 'acme_webserver':
  environment_name   => 'prod',
  port               => 80,
  worker_processes   => 'auto',
  worker_connections => 4096,
  keepalive_timeout  => 30,
  server_tokens      => false,    # never leak the version in prod
  vhosts             => {
    'portal' => {
      'server_name' => 'portal.acme.local',
      'docroot'     => '/var/www/portal',
      'default'     => true,
    },
    'api' => {
      'server_name' => 'api.acme.local',
      'docroot'     => '/var/www/api',
      'port'        => 80,
    },
  },
}
```

### Step 8 — Validate and apply

```bash
cd ~/puppet-modules
puppet parser validate acme_webserver/manifests/*.pp

# Dev
sudo puppet apply --modulepath=$HOME/puppet-modules \
  --noop acme_webserver/examples/dev.pp
sudo puppet apply --modulepath=$HOME/puppet-modules \
  acme_webserver/examples/dev.pp

curl -s http://localhost | grep -o "Environment: dev"
sudo nginx -T | grep -E "worker_connections|server_tokens"
```

```bash
# Prod profile on the same host - watch what changes
sudo puppet apply --modulepath=$HOME/puppet-modules \
  --noop acme_webserver/examples/prod.pp
sudo puppet apply --modulepath=$HOME/puppet-modules \
  acme_webserver/examples/prod.pp

curl -s http://localhost | grep -o "Environment: prod"
curl -s http://localhost/health
curl -sI http://localhost | grep -i server      # no version => server_tokens off
ls -l /etc/nginx/conf.d/
```

### Step 9 — Idempotence and parameter validation

```bash
# Run twice - second run must be clean
sudo puppet apply --modulepath=$HOME/puppet-modules acme_webserver/examples/prod.pp | \
  grep -c "changed" || echo "idempotent"

# Prove typed parameters fail fast
cat > /tmp/bad.pp <<'EOF'
class { 'acme_webserver': port => 99999 }
EOF
sudo puppet apply --modulepath=$HOME/puppet-modules /tmp/bad.pp
# Expect: parameter 'port' expects an Integer[1, 65535] value, got Integer[99999]
```

### Step 10 — `metadata.json` and versioning

```json
{
  "name": "acme-acme_webserver",
  "version": "1.0.0",
  "author": "ACME Platform Engineering",
  "summary": "ACME standard nginx web server",
  "license": "Apache-2.0",
  "source": "https://github.com/<you>/puppet-modules",
  "dependencies": [],
  "operatingsystem_support": [
    { "operatingsystem": "Ubuntu", "operatingsystemrelease": ["26.04"] }
  ],
  "requirements": [
    { "name": "puppet", "version_requirement": ">= 8.0.0 < 9.0.0" }
  ]
}
```

---

## 6. Acceptance criteria

- [ ] Module follows the standard directory layout
- [ ] Every parameter is typed and documented with `@param`
- [ ] No hard-coded paths/packages in `install`/`config`/`service` — all from `params.pp`
- [ ] `install -> config ~> service` ordering is explicit; config change restarts nginx
- [ ] Defined type creates multiple vhosts from a hash
- [ ] Three environments deploy from the **same module**, differing only in parameters
- [ ] Second apply reports no changes
- [ ] Invalid parameter value is rejected with a clear type error
- [ ] `purge => true` removes an unmanaged vhost file (test it)

---

## 7. Troubleshooting

| Symptom | Cause | Action |
|---|---|---|
| `Could not find class acme_webserver` | Wrong modulepath or directory name | `--modulepath` must point at the **parent** dir; module dir name must equal the module name |
| `Could not find template` | Template path wrong | `template('<module>/<file>.erb')`, file under `templates/` |
| `assert_private` error | Consumer included a private class directly | Only `include acme_webserver` / `class { 'acme_webserver': }` |
| Service doesn't restart after config change | Missing `notify`/`~>` | Use `~>` between config and service |
| Duplicate declaration for `File[docroot]` | vhost and config both declare the same path | Give vhosts their own docroot, or guard with `ensure_resource` |
| nginx fails to start | Bad template output | `sudo nginx -t`, then `sudo nginx -T` to read the rendered config |
| Parameter default `undef` errors | `inherits params` missing | `class x (...) inherits x::params { }` |
| Unmanaged vhost keeps coming back | `purge` not enabled on the dir | Set `purge => true, recurse => true` |

---

## 8. Cleanup

```bash
sudo systemctl disable --now nginx
sudo apt-get purge -y nginx
sudo rm -rf /etc/nginx /var/www/portal /var/www/api
```

---

## 9. Deliverables

1. Git repo `puppet-modules/acme_webserver` with full module structure and `metadata.json`.
2. `README.md` for the module: parameters table, examples, supported OS, changelog.
3. `MODULE-SOP.md` — design rules and the consumer upgrade procedure.
4. Evidence: `--noop` diffs for dev → prod, idempotent second run, type-validation failure.
5. Screenshots of the rendered pages for all three environments.

---

## 10. Interview talking points

* *"Why modules instead of one big manifest?"* — Reuse, a tested public API, independent
  versioning, and one place to fix a bug for every consumer.
* *"How do you support multiple OSes/environments from one module?"* — `params.pp` defaults
  selected by `os.family`, everything else exposed as typed parameters the consumer overrides.
* *"What does `contain` do?"* — Makes sub-class resources part of the containing class so
  ordering relationships (`->`, `~>`) apply to the whole class, not just its declaration.
* *"How do you stop someone depending on internals?"* — `assert_private()` on sub-classes;
  only `init.pp` is the public API, so internals can be refactored safely.
* *"What's the value of typed parameters?"* — Errors surface at catalog compile time with a clear
  message, instead of producing a broken config on 40 servers.
* *"Why `purge` the vhost directory?"* — So a file someone dropped on the server by hand is
  removed; otherwise unmanaged config silently survives.

**CV bullet:** *Built and published a reusable, typed, template-driven Puppet module for the
web-server fleet, replacing three divergent team-maintained manifests with a single versioned
module consumed across dev, staging and production via parameters.*
