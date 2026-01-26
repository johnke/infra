# Infrastructure Automation

Automation scripts and configuration for managing my personal server infrastructure.

## Servers Managed (for now)

- Pi-hole DNS server (Raspberry Pi)
- Web server running nginx (DigitalOcean VM)

## Repository Structure

- `ansible/` - Ansible playbooks and configuration
- (Future: terraform, docker-compose, scripts, etc.)

## Ansible Setup

### Prerequisites

- Ansible 2.9+
- Python 3.x
- SSH access to all servers
- pre-commit (optional but recommended)

Install Ansible:

```bash
mise use ansible@latest
# or pip if you have to:
pip install ansible
```

Install pre-commit for git hooks:

```bash
mise use pre-commit@latest
# or
brew install pre-commit
```

### Initial Setup

#### 0. Set Up Pre-Commit Hooks (Optional but Recommended)

Install the git hooks to prevent committing unencrypted vault files:

```bash
pre-commit install
```

This will check that vault files are encrypted before each commit.

#### 1. Create Inventory File

Copy the example inventory and fill in your actual server details:

```bash
cd ansible
cp inventory/hosts.example inventory/hosts
```

Edit `inventory/hosts` with your actual IPs and usernames. This file is gitignored.

#### 2. Set Up Ansible Vault

Get the vault password stored in 1password, and put it into the (.gitignore'd) `.vault_pass` file in the repo root.

```bash
echo "my-strong-vault-password" > .vault_pass
chmod 600 .vault_pass
```

#### 3. Create and Encrypt Secrets

Edit `vault.yml` with secrets, then encrypt it:

```bash
ansible-vault encrypt inventory/group_vars/all/vault.yml
```

To edit encrypted files later:

```bash
ansible-vault edit inventory/group_vars/all/vault.yml
```

#### 4. Configure Variables

- `ansible/inventory/group_vars/all/vars.yml` - Common settings
- `ansible/inventory/group_vars/pihole/vars.yml` - Pi-hole settings
- `ansible/inventory/group_vars/webservers/vars.yml` - Web server settings

### Usage

All commands should be run from the `ansible/` directory:

```bash
cd ansible
```

#### Test Connection

```bash
ansible all -m ping
```

#### Run Full Configuration

```bash
ansible-playbook playbooks/site.yml
```

#### Run Specific Server Configuration

```bash
# Pi-hole only
ansible-playbook playbooks/site.yml --limit pihole

# Web servers only
ansible-playbook playbooks/site.yml --limit webservers
```

#### Dry Run (Check Mode)

```bash
ansible-playbook playbooks/site.yml --check
```

## Security Best Practices

1. **Never commit sensitive files** - The `.gitignore` prevents this, but always verify before committing
2. **Use Ansible Vault** - All secrets should be encrypted
3. **Rotate vault password** periodically
4. **Use SSH keys** instead of passwords for server access
5. **Review diffs carefully** before pushing to git

### Managing Secrets

Add a new secret:

```bash
cd ansible
ansible-vault edit inventory/group_vars/all/vault.yml
```

Rekey (change vault password):

```bash
cd ansible
ansible-vault rekey inventory/group_vars/all/vault.yml
```

View encrypted file:

```bash
cd ansible
ansible-vault view inventory/group_vars/all/vault.yml
```

### Directory Structure

```
.
├── .vault_pass             # Vault password (gitignored)
├── ansible/
│   ├── ansible.cfg         # Ansible configuration
│   ├── inventory/
│   │   ├── hosts          # Server inventory (gitignored)
│   │   ├── hosts.example  # Template for hosts file
│   │   ├── group_vars/    # Group-specific variables
│   │   │   ├── all/      # Variables for all hosts
│   │   │   ├── pihole/   # Pi-hole specific
│   │   │   └── webservers/ # Web server specific
│   │   └── host_vars/    # Host-specific variables
│   ├── playbooks/        # Ansible playbooks
│   │   ├── site.yml     # Main playbook
│   │   ├── common.yml   # Common tasks
│   │   ├── pihole.yml   # Pi-hole tasks
│   │   ├── webserver.yml # Web server tasks
│   │   ├── handlers.yml # Event handlers
│   │   └── templates/   # Jinja2 templates
│   │       ├── nginx-site.conf.j2        # Static site template
│   │       ├── nginx-django-site.conf.j2 # Django site template
│   │       └── supervisor-django.conf.j2 # Supervisor config
│   ├── roles/           # Custom roles (if needed)
│   └── files/           # Static files to copy
└── README.md
```

## Troubleshooting

### Vault password errors

Ensure `.vault_pass` exists and contains the correct password.

### SSH connection issues

Check that:
- Your SSH key is added to the servers
- The correct username is in `ansible/inventory/hosts`
- The server IPs are correct

### Permission denied errors

Ensure your user has sudo privileges on the target servers.

### Adding New Servers

1. Add the server to `ansible/inventory/hosts`
2. Create `ansible/inventory/host_vars/[hostname]/` if needed
3. Add tasks to appropriate playbooks or create new ones
4. Test with `--limit` flag before running on all hosts
