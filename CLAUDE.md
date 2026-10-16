# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

Infrastructure automation for personal servers using Ansible. Manages two
active servers:
1. Raspberry Pi running Pi-hole (DNS/ad-blocker) -- Debian, `192.168.1.2`
2. `caesar.johnke.me` running nginx (web hosting) -- **OpenBSD**, `46.23.90.156`

A third host (`pi-apps`, a Pi 5 at `192.168.1.3` for Docker apps) is **built
but disabled** -- see "Parked: the pi-apps / linkding path" below.

The two hosts run different operating systems, which drives most of the
structure here. `common.yml` branches on `ansible_facts.os_family`; everything
under the `webservers` group is OpenBSD-only.

## Working Directory

**IMPORTANT**: All Ansible commands must be run from the `ansible/` directory:

```bash
cd ansible
```

The `ansible.cfg` file is located here and configures inventory paths, vault settings, and SSH behavior.

## Common Commands

### Testing and Deployment

```bash
# Test connection to all servers
ansible all -m ping

# Dry run (check mode) - see what would change without making changes
ansible-playbook playbooks/site.yml --check

# Run on specific server group only
ansible-playbook playbooks/site.yml --limit pihole
ansible-playbook playbooks/site.yml --limit webservers

# Deploy to all servers
ansible-playbook playbooks/site.yml
```

### Vault Management

```bash
# Edit encrypted secrets (uses .vault_pass automatically)
ansible-vault edit inventory/group_vars/all/vault.yml

# View encrypted file without editing
ansible-vault view inventory/group_vars/all/vault.yml

# Encrypt a new vault file
ansible-vault encrypt path/to/vault.yml

# Change vault password
ansible-vault rekey inventory/group_vars/all/vault.yml
```

## Architecture Overview

### Playbook Structure

The main playbook (`playbooks/site.yml`) orchestrates three plays:

1. **Configure all servers** - Runs `common.yml` on all hosts
   - System updates, timezone, common packages
   - UFW firewall configuration
   - SSH access rules

2. **Configure Pi-hole** - Runs `pihole.yml` on pihole group
   - Pi-hole installation and configuration
   - DNS port rules (53 TCP/UDP)
   - Web interface setup (port 80)

3. **Configure application hosts** - Runs `apps.yml` on apps group (Debian)
   - WireGuard client dialling out to caesar
   - Docker installation
   - linkding via Docker Compose, bound to the tunnel address

4. **Configure web servers** - Runs `webserver.yml` on webservers group (OpenBSD)
   - Nginx installation via `openbsd_pkg`
   - Multi-site configuration via templates into `/etc/nginx/conf.d/`
   - TLS certificates installed by hand (Cloudflare Origin certs)
   - Django apps supervised by `rc.d(8)` / `rcctl`
   - WireGuard hub interface (`wg0`) for tunnelled app hosts

**Pattern**: Uses `include_tasks` for composition rather than roles. Handlers are imported per-play.

### OpenBSD specifics (webservers)

These differ from the Debian conventions and are easy to get wrong:

| Concern | Debian (old) | OpenBSD (current) |
| --- | --- | --- |
| Packages | `apt` | `community.general.openbsd_pkg` |
| Privilege escalation | `sudo` | `doas` (set per-group in `inventory/hosts`) |
| Python interpreter | `/usr/bin/python3` | `/usr/local/bin/python3` |
| Firewall | `ufw`, rule-at-a-time | `pf`, whole ruleset from `pf.conf.j2` |
| Web user | `www-data` | `www` |
| Site configs | `sites-available` + symlinks | one file per site in `/etc/nginx/conf.d/` |
| TLS | `certbot` | manual Cloudflare Origin certs |
| Process supervision | `supervisor` | `rc.d(8)` scripts + `rcctl` |
| File group | `root` | `wheel` |

**nginx runs chrooted in `/var/www`,** and this cuts two different ways
depending on *when* nginx opens the path:

- **Document roots are chroot-relative.** They are resolved after the chroot,
  so `/var/www/htdocs/johnke.me` must be written `/htdocs/johnke.me`. The
  templates strip the prefix with
  `item.root | regex_replace('^' ~ chroot_base, '')`.
- **Log paths are real, absolute host paths** (`{{ nginx_log_dir }}`, i.e.
  `/var/www/logs/...`). nginx opens its log files as root *before* chrooting,
  so the real path is correct — and it is also the only form that works with
  `nginx -t`, which runs outside the chroot and would fail with ENOENT on a
  chroot-relative `/logs/...`. (`nginx -t -p /var/www` does **not** fix that;
  `-p` only affects relative paths.)

Ansible tasks that create these directories always use real paths.

**Docker is not available on OpenBSD.** Containerised apps run on `pi-apps`
at home and are published by caesar over a WireGuard tunnel -- see below.

### Parked: the pi-apps / linkding path

**Status: fully implemented but switched off (2026-10-08).** Everything below
describes code that exists in the repo and does nothing. It is gated by three
independent switches, so no single accidental edit turns it on:

| Switch | Location | Current |
| --- | --- | --- |
| `pi-apps` host | `inventory/hosts` | commented out (apps group empty) |
| `apps_enabled` | `group_vars/apps/vars.yml` | `false` |
| `linkding_enabled` | `group_vars/apps/vars.yml` | `false` |
| `wg_enabled` | `group_vars/all` + `webservers` | `false` |
| `bookmarks.johnke.me` | `group_vars/webservers/vars.yml` | commented out of `nginx_sites` |
| `51820/udp` | `pf_allowed_udp_ports` | `[]` (closed) |

While parked, the `vault_wg_*` and `vault_linkding_superuser_password` secrets
are **not needed** -- the templates referencing them are never rendered.

To re-enable, follow the numbered checklist at the top of
`inventory/group_vars/apps/vars.yml`.

#### The design (for when we revisit)

`pi-apps` is on a home connection behind **CGNAT**, so it has no routable
public address and nothing can connect *to* it. The tunnel works because the
Pi always dials out:

```
internet -> caesar :443 (TLS) -> wg0 10.99.0.2:9090 -> linkding (Docker, pi-apps)
                 ^                                          |
                 +--- Pi initiates, keepalive holds NAT -----+
```

Key points, all of which are load-bearing:

- **No `Endpoint` for the Pi in caesar's config.** caesar cannot know the Pi's
  address; `wg(4)` learns it from the authenticated handshake, which is also
  why Virgin Media's dynamic IP does not matter.
- **`PersistentKeepalive = 25` on the Pi only.** NAT mappings expire when idle;
  without this, caesar silently loses the ability to reach the Pi. caesar needs
  no keepalive because it never initiates.
- **`AllowedIPs = 10.99.0.0/24`**, not `0.0.0.0/0`. This is a point-to-point
  link, not a default-route VPN -- the Pi's normal traffic and Pi-hole DNS are
  untouched.
- **MTU 1420.** WireGuard adds ~60 bytes; the classic symptom of getting this
  wrong is small pages loading while large ones hang.
- **`51820/udp` must be open in pf** (via `pf_allowed_udp_ports`).
- **Docker is ordered after `wg-quick@wg0`** by a systemd drop-in, because
  linkding binds to `10.99.0.2` and that address does not exist until the
  tunnel is up. Without the override, Docker loses the race on boot.

On OpenBSD the interface is `/etc/hostname.wg0` (wg is in base, no package,
no `wg-quick`); on the Pi it is `/etc/wireguard/wg0.conf` + `wg-quick@wg0`.

**Accepted trade-off**: bookmarks are unavailable when the home connection or
the Pi is down. caesar serves a readable 503 in that case
(`proxy_custom_error_page: true`) rather than nginx's default 502.

### Inventory Organization

- **Hosts file**: `inventory/hosts` (gitignored) - contains actual IPs and usernames
- **Host aliases**: `pi` (pihole), `web` (webserver) - never commit real hostnames
- **Group structure**: `pihole`, `apps`, `webservers`, `all`

### Variable Hierarchy

Variables are organized by scope:

```
inventory/
├── group_vars/
│   ├── all/
│   │   ├── vars.yml       # Non-sensitive globals (timezone, ssh_port, etc.)
│   │   └── vault.yml      # Encrypted secrets (passwords, emails, tokens)
│   ├── pihole/
│   │   └── vars.yml       # Pi-hole specific (DNS servers, blocklists)
│   └── webservers/
│       └── vars.yml       # Nginx sites configuration
└── host_vars/             # Host-specific overrides (currently unused)
```

**Important variables:**
- `nginx_sites` (webservers/vars.yml) - List of sites with domains, SSL paths, custom config
- `chroot_base` / `web_root_base` - `/var/www` and `/var/www/htdocs`
- `web_user` / `web_group` - `www` / `www`
- `pf_allowed_tcp_ports` - extra inbound ports; SSH has its own rate-limited rule, so do not list `ssh_port` here
- `vault_pihole_password` (vault.yml) - Pi-hole web interface password

### Content Deployment

Web roots are owned by the **deploy user** (`johnke`) with group `www`,
mode `0755`:

```
/var/www/htdocs/johnke.me   johnke:www  0755
```

nginx runs as `www` and only ever *reads* content, so it gets what it needs
through the group bits. Making `www` the owner instead looks tidier but
breaks CI: on BSD, setting a directory's setgid bit requires **ownership**,
not just group membership, so `rsync --chmod=D2775` fails with EPERM on every
`www`-owned directory. Deploy-user ownership avoids the whole problem.

GitHub Actions deploys Hugo output with:

```bash
rsync -aze ssh --delete --omit-dir-times --no-owner --no-group \
  --chmod=D755,F644 ./public/ johnke@$SSH_HOST:/var/www/htdocs/johnke.me/
```

- `--chmod` sets modes explicitly so a new directory never inherits a
  restrictive umask from the CI runner. (`--chmod` and `--no-perms` conflict
  -- use one or the other.)
- `--no-owner --no-group` because the deploy user cannot chown.
- `SSH_HOST` must **not** be a Cloudflare-proxied name -- proxied DNS returns
  Cloudflare's addresses and SSH will not connect. Use the origin IP or an
  unproxied record.

### Nginx Site Management

Each site gets one config file in `/etc/nginx/conf.d/<name>.conf`, included by
`/etc/nginx/nginx.conf` (itself generated from `nginx.conf.j2`). There is no
sites-available/sites-enabled symlink dance on OpenBSD. Two site types are
supported: `static`, `django`, and `proxy`.

Shared TLS settings (protocols, ciphers, session cache, stapling, dhparam)
live once in `nginx.conf`; per-site configs only set `ssl_certificate` and
`ssl_certificate_key`.

#### Static Sites

1. **Configuration**: Edit `inventory/group_vars/webservers/vars.yml`
2. **Template**: `playbooks/templates/nginx-site.conf.j2` generates configs
3. **Deployment**: Playbook writes `/etc/nginx/conf.d/<name>.conf`
4. **Handler**: Triggers nginx reload on config changes

**Static site structure:**
```yaml
nginx_sites:
  - name: example.com
    site_type: static  # or omit (default is static)
    server_names:
      - example.com
      - www.example.com
    root: "{{ web_root_base }}/example.com"   # real path; template strips the chroot prefix
    ssl_enabled: true
    ssl_certificate: /etc/ssl/example.com.fullchain.pem
    ssl_certificate_key: /etc/ssl/private/example.com.key
    custom_config: |
      # Optional custom nginx directives
```

#### Django Sites

For Django apps with Gunicorn, supervised by `rc.d(8)`:

1. **Configuration**: Edit `inventory/group_vars/webservers/vars.yml` with `site_type: django`
2. **Templates**:
   - `playbooks/templates/nginx-django-site.conf.j2` - Nginx reverse proxy config
   - `playbooks/templates/rc-django.j2` - rc.d service script
3. **Deployment**: Creates nginx config, an `/etc/rc.d/<service_name>` script, and log directories
4. **Service**: Enabled and started via `rcctl` (the `ansible.builtin.service` module drives this)

**Django site structure:**
```yaml
nginx_sites:
  - name: app.example.com
    site_type: django  # Required for Django sites
    server_names:
      - app.example.com
      - www.app.example.com
    root: "{{ web_root_base }}/app.example.com"
    ssl_enabled: true
    ssl_certificate: /etc/ssl/app.example.com.fullchain.pem
    ssl_certificate_key: /etc/ssl/private/app.example.com.key
    gunicorn_port: 8000
    gunicorn_workers: 3           # optional, defaults to 3
    wsgi_module: config.wsgi:application
    client_max_body_size: 10M
    service_name: app_example     # rc.d script name -- no dots allowed
    rc_environment:               # Optional env vars
      DJANGO_SETTINGS_MODULE: config.settings.production
```

The app is expected to have a virtualenv at `<root>/venv` with gunicorn
installed; the rc.d script invokes `<root>/venv/bin/gunicorn` directly.

#### Reverse-Proxy Sites

For apps running elsewhere -- including on `pi-apps` across the WireGuard
tunnel. Template: `nginx-proxy-site.conf.j2`.

```yaml
nginx_sites:
  - name: app.example.com
    site_type: proxy
    server_names:
      - app.example.com
    ssl_enabled: true
    ssl_certificate: /etc/ssl/app.example.com.fullchain.pem
    ssl_certificate_key: /etc/ssl/private/app.example.com.key
    proxy_host: "{{ wg_pi_apps_ip }}"   # defaults to 127.0.0.1
    proxy_port: 9090
    client_max_body_size: 50M
    proxy_custom_error_page: true       # readable 503 when upstream is down
    proxy_connect_timeout: 5s           # fail fast over a flaky link
```

WebSocket upgrades use the `$connection_upgrade` map defined in `nginx.conf`
rather than a hardcoded `Connection: upgrade`, which would break keepalive on
ordinary requests.

**Note**: Templates must be in `playbooks/templates/` directory since playbooks reference them with relative paths.

## Security Architecture

### Secrets Management

**Vault password**: Stored in `.vault_pass` at repo root (gitignored). Ansible reads it automatically via `ansible.cfg`.

**Never commit:**
- `inventory/hosts` - Contains real IPs and usernames
- Any `vault.yml` files - Encrypted secrets
- `.vault_pass` - Vault password file
- SSH keys, `.env` files

All protected by comprehensive `.gitignore`.

### Adding New Secrets

1. Edit the vault: `ansible-vault edit inventory/group_vars/all/vault.yml`
2. Add variable with `vault_` prefix: `vault_new_secret: "value"`
3. Reference in vars files: `some_setting: "{{ vault_new_secret }}"`

## Important Patterns

### Adding a New Site

1. Edit `ansible/inventory/group_vars/webservers/vars.yml`
2. Add entry to `nginx_sites` list
3. Test: `ansible-playbook playbooks/site.yml --limit webservers --check`
4. Deploy: `ansible-playbook playbooks/site.yml --limit webservers`

### Modifying Existing Playbooks

When editing playbooks (`common.yml`, `pihole.yml`, `webserver.yml`):
- Tasks that change configs should notify handlers (e.g., `notify: reload nginx`)
- Use `when` conditionals for OS-specific tasks
- Always use fully qualified module names (`ansible.builtin.apt` not `apt`)
- `common.yml` runs on both OSes -- guard every task with `ansible_facts["os_family"]`
- `webserver.yml` is OpenBSD-only and needs no such guards

### Certificates

**Certificates are managed by hand, not by Ansible.** `johnke.me` uses a
**Cloudflare Origin Certificate** (issued 2026-10-08, valid to **2041**,
covering `johnke.me` and `*.johnke.me`). There is no ACME client, no renewal
cron, and no `certbot`/`acme-client` on the host -- Cloudflare terminates
public TLS and this cert only secures the Cloudflare-to-origin hop.

Consequences worth knowing:

- An Origin cert is **only trusted by Cloudflare**, not by browsers. Reaching
  caesar's IP directly over HTTPS gives a certificate warning. Cloudflare's
  SSL/TLS mode must be **Full (strict)**.
- `*.johnke.me` does **not** cover `johnke.wtf` (different registrable
  domain) and a wildcard does not match the apex, which is why the SAN lists
  both `johnke.me` and `*.johnke.me`. `johnke.wtf` is still in `server_names`
  but has no certificate of its own.
- Nothing expires for ~15 years, so there is no renewal to monitor.

Expected paths (already set in `nginx_sites`):

```
/etc/ssl/johnke.me.fullchain.pem      0444 root:wheel
/etc/ssl/private/johnke.me.key        0400 root:wheel
```

To install or replace one, on the host (never commit key material, and never
paste a private key into a chat or a file in this repo):

```bash
doas tee /etc/ssl/johnke.me.fullchain.pem > /dev/null <<'EOF'
<certificate>
EOF
doas chmod 0444 /etc/ssl/johnke.me.fullchain.pem

doas tee /etc/ssl/private/johnke.me.key > /dev/null <<'EOF'
<private key>
EOF
doas chown root:wheel /etc/ssl/private/johnke.me.key
doas chmod 0400 /etc/ssl/private/johnke.me.key

# the two hashes must match
doas openssl x509 -in /etc/ssl/johnke.me.fullchain.pem -noout -modulus | openssl md5
doas openssl rsa  -in /etc/ssl/private/johnke.me.key   -noout -modulus | openssl md5
```

**Bootstrap ordering.** nginx refuses to start when a configured
`ssl_certificate` file is missing. `webserver.yml` stats each certificate
first and deploys sites that lack one as **HTTP-only**, so nginx always
starts; installing the cert and re-running the playbook picks it up and
deploys the HTTPS config. The site templates gate their HTTPS block on
`item.ssl_enabled and has_cert`, from the `sites_with_certs` fact.

## Installation Method

Ansible installed via `mise`:
```bash
mise use ansible@latest
```

Or fallback to pip:
```bash
pip install ansible
```

## Known Issues

- SSH ControlMaster can cause hanging - removed from config
- Timer/profile_tasks callbacks may not exit cleanly - currently using minimal callbacks
