# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

Infrastructure automation for personal servers using Ansible. Manages two main servers:
1. Raspberry Pi running Pi-hole (DNS/ad-blocker)
2. DigitalOcean VM running nginx (web hosting)

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

3. **Configure web servers** - Runs `webserver.yml` on webservers group
   - Nginx and Certbot installation
   - Multi-site configuration via templates
   - SSL certificate management
   - HTTP/HTTPS firewall rules

**Pattern**: Uses `include_tasks` for composition rather than roles. Handlers are imported per-play.

### Inventory Organization

- **Hosts file**: `inventory/hosts` (gitignored) - contains actual IPs and usernames
- **Host aliases**: `pi` (pihole), `web` (webserver) - never commit real hostnames
- **Group structure**: `pihole`, `webservers`, `all`

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
- `vault_certbot_email` (vault.yml) - Email for Let's Encrypt
- `vault_pihole_password` (vault.yml) - Pi-hole web interface password

### Nginx Site Management

Nginx sites use the standard sites-available/sites-enabled pattern. Two site types are supported:

#### Static Sites

1. **Configuration**: Edit `inventory/group_vars/webservers/vars.yml`
2. **Template**: `playbooks/templates/nginx-site.conf.j2` generates configs
3. **Deployment**: Playbook creates files in `/etc/nginx/sites-available/` and symlinks to `/etc/nginx/sites-enabled/`
4. **Handler**: Triggers nginx reload on config changes

**Static site structure:**
```yaml
nginx_sites:
  - name: example.com
    site_type: static  # or omit (default is static)
    server_names:
      - example.com
      - www.example.com
    root: /var/www/example
    ssl_enabled: true
    ssl_certificate: /etc/letsencrypt/live/example.com/fullchain.pem
    ssl_certificate_key: /etc/letsencrypt/live/example.com/privkey.pem
    custom_config: |
      # Optional custom nginx directives
```

#### Django Sites

For Django apps with Gunicorn and Supervisor:

1. **Configuration**: Edit `inventory/group_vars/webservers/vars.yml` with `site_type: django`
2. **Templates**:
   - `playbooks/templates/nginx-django-site.conf.j2` - Nginx reverse proxy config
   - `playbooks/templates/supervisor-django.conf.j2` - Supervisor config
3. **Deployment**: Creates nginx config, supervisor config in `/etc/supervisor/conf.d/`, and log directories
4. **Handlers**: Triggers nginx reload and supervisor update

**Django site structure:**
```yaml
nginx_sites:
  - name: app.example.com
    site_type: django  # Required for Django sites
    server_names:
      - app.example.com
      - www.app.example.com
    root: /var/www/app.example.com
    ssl_enabled: true
    use_certbot_ssl_config: true  # Use Certbot's SSL config files
    ssl_certificate: /etc/letsencrypt/live/app.example.com/fullchain.pem
    ssl_certificate_key: /etc/letsencrypt/live/app.example.com/privkey.pem
    gunicorn_port: 8000
    wsgi_module: config.wsgi:application
    client_max_body_size: 10M
    supervisor_environment:  # Optional env vars
      DJANGO_SETTINGS_MODULE: config.settings.production
```

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

### SSL Certificates

Certbot is installed and auto-renewal is enabled via systemd timer. The playbook does NOT automatically request new certificates (`certbot_create_if_missing: false`). SSL certificates must be manually obtained or already present.

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
