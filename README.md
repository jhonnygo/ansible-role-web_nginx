# Ansible Role: **web_nginx**
[![Style: ansible-lint](https://img.shields.io/badge/style-ansible--lint-green)](#lint)
[![License: MIT](https://img.shields.io/badge/license-MIT-informational)](LICENSE)

> <img src="img/nginx.jpg?raw=true" alt="NGINX Logo" height="100px"/>

<br/>

Ansible role to **install and configure NGINX with declarative vhosts** on Debian/Ubuntu:

- Installs `nginx` and manages its **service state** (`started`/`stopped`) and **enablement** on boot.
- Declarative **server blocks (vhosts)** via variables; rendered from clean **Jinja2 templates** (HTTP/HTTPS/catch‑all).
- Creates **docroots** (0755), seeds optional **index.html** smoke page, and per‑vhost **access/error logs**.
- Enables sites with symlinks in `sites-enabled/`, validates with `nginx -t`, and **reloads** on changes.
- Optional **catch‑all** site (modes `403`, `404`, or `redirect`) as `default_server` for unknown hosts.
- Optional **HTTPS** per vhost (self‑signed or provided cert/key) with separate `-ssl.conf` files.
- Optional **purge of unmanaged vhosts** (safe excludes). Filenames are normalized without `.conf` to compare.
- **conf.d** toggles for gzip/proxy — duplicate‑safe (this role does **not** force `gzip on;`). 
- Idempotent; first run works even with `--check` thanks to guarded install tasks.

---

## Contents
- [Compatibility](#compatibility)
- [Quick Start](#quick-start)
- [Role Variables](#role-variables)
- [Usage Examples](#usage-examples)
- [Enable Reverse Proxy](#enable-reverse-proxy)
- [Test Matrix](#test-matrix)
- [Lint](#lint)
- [Links](#links)

---

## Compatibility

**OS**: Ubuntu 20.04/22.04/24.04; Debian 11/12  
**Ansible**: 2.12+

---

## Quick Start

```ini
# ansible.cfg
[defaults]
roles_path = ./roles:~/.ansible/roles
```

```yaml
# playbooks/web_nginx.yml
- hosts: all
  become: true
  roles:
    - role: web_nginx
```

**Variables (example, group_vars/role_app.yml):**

```yaml
web_nginx_vhosts:
  - server_name: "stage.site1.com"
    filename: "100-site1.conf"
    root: /var/www/site1
    locations:
      - path: "/"
        try_files: "$uri $uri/ /index.html"

  - server_name: "api.site2.com"
    filename: "110-api.conf"
    root: /var/www/empty
    locations:
      - path: /
        proxy_pass: "http://127.0.0.1:8080"
        proxy_set_headers:
          Host: "$host"
          X-Forwarded-For: "$proxy_add_x_forwarded_for"
    https:
      enabled: true
      self_signed: true
```

---

## Role Variables

**Main toggles**

```yaml
web_nginx_state: started                 # started|stopped
web_nginx_enabled: true
web_nginx_listen_port: 80
web_nginx_manage_catch_all: true
web_nginx_remove_distro_default: true
web_nginx_purge_unlisted_vhosts: false
web_nginx_purge_exclude_names: ["default","000-catch-all"]
web_nginx_gzip: true
web_nginx_proxy: true
```

**Catch‑all**

```yaml
web_nginx_catch_all_mode: "403"         # 403|404|redirect
web_nginx_catch_all_redirect_to: "https://example.org"
web_nginx_catch_all_docroot: "/var/www/empty"
web_nginx_catch_all_filename: "000-catch-all.conf"
```

**Paths & Ownership**

```yaml
web_nginx_sites_available: /etc/nginx/sites-available
web_nginx_sites_enabled:   /etc/nginx/sites-enabled
web_nginx_log_dir:         /var/log/nginx
web_nginx_docroot_default_owner: "www-data"
web_nginx_docroot_default_group: "www-data"
web_nginx_docroot_default_mode:  "0755"
web_nginx_default_index: "index.html index.htm"
```

**Vhost structure** (per item)

```yaml
- server_name: "app.example.com"
  filename: "100-app.conf"          # optional; derived from server_name if omitted
  root: "/var/www/app"
  index: "index.html index.htm"
  extra_server_names: ["www.app.example.com"]
  listen: "80"                       # accepts "IP:port" or port
  access_log: "/var/log/nginx/app-access.log"
  error_log:  "/var/log/nginx/app-error.log"
  locations:
    - path: "/"
      try_files: "$uri $uri/ /index.html"
    - path: "/api/"
      proxy_pass: "http://127.0.0.1:8080"
      proxy_set_headers:
        Host: "$host"
        X-Forwarded-For: "$proxy_add_x_forwarded_for"
  https:
    enabled: false
    listen: "443 ssl http2"
    certificate: "/etc/ssl/certs/app.crt"
    certificate_key: "/etc/ssl/private/app.key"
    self_signed: false
    self_signed_common_name: "{{ server_name }}"
```

---

## Usage Examples

**Minimal static site**
```yaml
- hosts: role_app
  become: true
  roles:
    - role: web_nginx
      vars:
        web_nginx_vhosts:
          - server_name: "stage.site1.com"
            filename: "100-site1.conf"
            root: /var/www/site1
            locations:
              - path: "/"
                try_files: "$uri $uri/ /index.html"
```

**Reverse proxy**
```yaml
web_nginx_vhosts:
  - server_name: "api.example.com"
    filename: "110-api.conf"
    root: /var/www/empty
    locations:
      - path: /
        proxy_pass: "http://127.0.0.1:8080"
        proxy_set_headers:
          Host: "$host"
          X-Forwarded-For: "$proxy_add_x_forwarded_for"
```

**HTTPS (self‑signed)**
```yaml
web_nginx_vhosts:
  - server_name: "secure.example.com"
    filename: "120-secure.conf"
    root: /var/www/secure
    locations:
      - path: "/"
        try_files: "$uri $uri/ /index.html"
    https:
      enabled: true
      self_signed: true
```

---

## Enable Reverse Proxy

The role already provides safe proxy defaults (`conf.d/proxy.conf`) when `web_nginx_proxy: true` (default). To enable a reverse proxy for a vhost, add a `location` with `proxy_pass`:

```yaml
web_nginx_vhosts:
  - server_name: "api.example.com"
    filename: "110-api.conf"
    root: /var/www/empty
    locations:
      - path: /
        proxy_pass: "http://127.0.0.1:8080"
        proxy_set_headers:
          Host: "$host"
          X-Forwarded-For: "$proxy_add_x_forwarded_for"
```

**WebSockets / HTTP/1.1**
```yaml
locations:
  - path: /socket
    proxy_pass: "http://127.0.0.1:3000"
    proxy_set_headers:
      Host: "$host"
      X-Forwarded-For: "$proxy_add_x_forwarded_for"
      Upgrade: "$http_upgrade"
      Connection: "upgrade"
```

**Front-end HTTPS with back-end HTTP**
```yaml
web_nginx_vhosts:
  - server_name: "secure.example.com"
    filename: "120-secure.conf"
    root: /var/www/empty
    locations:
      - path: /
        proxy_pass: "http://127.0.0.1:8080"
        proxy_set_headers:
          Host: "$host"
          X-Forwarded-For: "$proxy_add_x_forwarded_for"
    https:
      enabled: true
      self_signed: true      # or provide your cert/key paths
```

---

## Test Matrix

| Distro / Version | Arch | NGINX | Python | Notes |
|---|---:|---:|---:|---|
| Ubuntu 24.04 (Noble)   | x86_64 | distro | 3.12 | ✅ Default server, vhost OK |
| Ubuntu 22.04 (Jammy)   | x86_64 | distro | 3.10 | ✅ |
| Ubuntu 20.04 (Focal)   | x86_64 | distro | 3.8  | ✅ |
| Debian 12 (Bookworm)   | x86_64 | distro | 3.11 | ✅ |
| Debian 11 (Bullseye)   | x86_64 | distro | 3.9  | ✅ |

> Molecule scenario exercises: package install, service active, port 80 listening, vhost responds at `/`.

---

## Lint

```bash
ansible-lint roles/web_nginx
yamllint roles/web_nginx
```

---

## Links

- [**GitHub (JhonnyGO):**](https://github.com/jhonnygo/ansible-role-web_nginx)
- [**Ansible Galaxy (jhonnyGO):**](https://galaxy.ansible.com/ui/standalone/namespaces/24365)

<br/>

<img src="img/happy-coding.jpg?raw=true" alt="Footer Logo" />

## License & Author

**License:** MIT  
**Author:** Jhonny Alexander (JhonnyGO) — <support@jhoncytech.com>
