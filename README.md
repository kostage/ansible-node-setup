# Ansible Repository for Server Management

This repository uses a modular role architecture for managing web services including Grafana and Headscale. The architecture eliminates code duplication and provides consistent configuration patterns across all services.

## SSH Connection Troubleshooting

If you cannot SSH to a server after running a playbook (e.g., "Too many authentication failures"), this is typically caused by having too many SSH keys loaded in your SSH agent. The SSH client tries all keys until the server disconnects after too many failed attempts.

### Solution

Add `IdentitiesOnly=yes` to your SSH command to only use the specified identity file:

```bash
ssh -o IdentitiesOnly=yes -i /path/to/your_key user@host
```

## Quick Start

### Python Virtual Environment Setup

```bash
# Install pyenv and virtualenv plugin (macOS)
brew install pyenv
brew install pyenv-virtualenv

# Install Python 3.11.5
pyenv install 3.11.5

# Create ansible virtualenv
pyenv virtualenv 3.11.5 ansible

# Activate the ansible environment
pyenv shell ansible
```

### Install Python Dependencies

```bash
pyenv shell ansible
pip install -r requirements.txt
```

### Install Ansible Collections and Roles

```bash
pyenv shell ansible
ansible-galaxy collection install -r requirements.yml
ansible-galaxy role install -r requirements.yml
```

### Localhost Setup

```bash
# Install GNU tar for macOS compatibility
brew install gnu-tar

# Install KeePass lookup plugin locally
cp ~/.ansible/collections/ansible_collections/viczem/keepass/plugins/lookup/keepass.py ~/.ansible/plugins/lookup
```

**Note:** The Grafana role now downloads the GPG key directly on the target server, so there's no need for local workarounds.

## Complete Environment Setup from Scratch

If you need to create a completely fresh ansible environment:

```bash
# 1. Remove existing ansible virtualenv (if it exists)
pyenv uninstall -f ansible

# 2. Recreate the virtualenv
pyenv virtualenv 3.11.5 ansible

# 3. Activate the environment
pyenv shell ansible

# 4. Upgrade pip
pip install --upgrade pip

# 5. Install Python dependencies
pip install -r requirements.txt

# 6. Verify ansible installation
ansible --version

# 7. Install Ansible collections and roles
ansible-galaxy collection install -r requirements.yml
ansible-galaxy role install -r requirements.yml

# 8. Verify collection installation
ansible-galaxy collection list

# 9. Setup localhost KeePass plugin
mkdir -p ~/.ansible/plugins/lookup
cp ~/.ansible/collections/ansible_collections/viczem/keepass/plugins/lookup/keepass.py ~/.ansible/plugins/lookup/

# 10. Verify environment
ansible-galaxy collection list | grep -E "(grafana|community|ansible.posix|keepass)"
```

## Current Environment Versions

### Python Packages (requirements.txt)
- **ansible-core**: 2.20.4
- **bcrypt**: 4.1.3
- **passlib**: 1.7.4
- **pykeepass**: 4.1.1.post1

### Ansible Collections (requirements.yml)
- **ansible.posix**: 2.1.0
- **community.general**: 12.5.0
- **community.crypto**: 3.1.1
- **grafana.grafana**: >=3.0.0 (currently 6.0.6)
- **prometheus.prometheus**: (latest)
- **viczem.keepass**: (latest)

### Ansible Roles (requirements.yml)
- **geerlingguy.java**
- **nginxinc.nginx**
- **nginxinc.nginx_config**
- **geerlingguy.certbot**

### Running Playbooks

```bash
export OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES
env no_proxy='*' ansible-playbook --ask-vault-password playbooks/grafana.yml -i hosts.yml
```

## Architecture Overview

### Flat Role Design

This repository uses a completely flat architecture with no loops or aggregators. Every setup step is explicitly visible in the playbooks:

#### Core Component Roles
- **`nginx_config`** - Universal NGINX configuration template for any service
- **`certbot`** - SSL certificate management using certbot
- **`oauth2_proxy`** - OAuth2 proxy service management

#### Service-Specific Roles
- **`headscale_service`** - Headscale VPN coordination server
- **`grafana_service`** - Grafana monitoring dashboards using grafana.grafana collection with native installation
- **`grafana_container`** - Grafana monitoring dashboards using Podman with systemd system service (legacy)

### Flat Variable Structure

All services use direct variables with no nested structures:

```yaml
# Service domains (direct variables)
headscale_domains:
  - "headscale.example.com"
  - "www.headscale.example.com"

grafana_domains:
  - "dashboard.example.com"

# Grafana authentication (using grafana_ini.security structure)
grafana_ini:
  security:
    admin_user: "admin_username"
    admin_password: "admin_password"
    cookie_secure: true
    cookie_samesite: lax
    disable_gravatar: true

# SSL configuration (single variable)
ssl_email: "admin@example.com"
```

### Features

#### Universal NGINX Configuration
- Single template works for all services
- Conditional OAuth2 authentication blocks
- Automatic SSL certificate integration
- Standardized security headers
- WebSocket support for services like Headscale

#### OAuth2 Proxy Management
- Multiple proxy instances per host
- Template-based configuration generation
- Systemd service creation and management

#### SSL Certificate Management
- Automated certbot integration
- Multi-domain certificate support
- Automatic renewal setup

## Available Playbooks

### grafana.yml
- **Purpose**: Complete monitoring stack setup
- **Services**: Grafana, Prometheus, Node Exporter, Alertmanager
- **Architecture**: Legacy (will be migrated to new service architecture)

### headplane.yml
- **Purpose**: Headscale + Grafana on single server
- **Services**: Headscale, Grafana, Prometheus, Alertmanager, Node Exporter
- **Architecture**: Completely flat structure with explicit steps
- **Features**: Every setup step visible, no abstraction layers
- **Authentication**: Grafana uses basic authentication (same password as non-root user)

### test_grafana.yml
- **Purpose**: Test Grafana deployment on homester server
- **Services**: User creation, packages, firewall, NGINX, Grafana container
- **Target**: `vpn` host group (homester)
- **Features**:
  - Complete prerequisite setup (user, apt, firewall, NGINX)
  - Grafana container using Podman with systemd service
  - SSL certificate integration
  - Based on headplane.yml but monitoring-only (no headscale/headplane)

### test_grafana_container.yml
- **Purpose**: Test Grafana deployment using Podman with systemd system service
- **Services**: Grafana container with Podman, optional NGINX reverse proxy
- **Requirements**: Podman 5.0+ (will be installed automatically)
- **Features**:
  - System service deployment
  - Data directory for persistence
  - System-scoped systemd service
  - Health checks and automatic restarts
  - Optional SSL certificates and NGINX reverse proxy

## Podman Systemd Service for Grafana

This repository now supports deploying Grafana using Podman with systemd system service, which provides container management with standard systemd integration.

### Prerequisites

The role will automatically install Podman 5.0+ if not present. The following will be configured:
- Podman and required dependencies
- Grafana system user (UID 472)
- Data directory at `/var/lib/grafana`
- Systemd service file in `/etc/systemd/system/grafana.service`
- System-scoped systemd service

### Role Variables

Key variables for `grafana_container` role:

```yaml
# Grafana administration (use KeePass lookups in production)
grafana_admin_user: "admin"
grafana_admin_password: "changeme"

# Domains for SSL certificates (optional)
grafana_domains:
  - "grafana.example.com"
  - "www.grafana.example.com"

# Grafana configuration (grafana.ini format)
grafana_ini_config:
  security:
    admin_user: "{{ grafana_admin_user }}"
    admin_password: "{{ grafana_admin_password }}"
    cookie_secure: true
    cookie_samesite: "lax"
    disable_gravatar: true
```

Note: Configuration is passed to the container via environment variables, not via grafana.ini file.

### Testing the Grafana Container Role

1. **Create inventory file** (e.g., `test_inventory.yml`):
```yaml
test_servers:
  hosts:
    your-test-server.example.com:
      ansible_user: your_ssh_user
      ansible_become: true
```

2. **Configure variables** in `host_vars/your-test-server.yml`:
```yaml
# Use KeePass for credentials in production
grafana_admin_user: "admin"
grafana_admin_password: "secure_password_here"

grafana_domains:
  - "grafana.example.com"

letsencrypt_email: "admin@example.com"
```

3. **Run the test playbook**:
```bash
export OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES
env no_proxy='*' ansible-playbook playbooks/test_grafana_container.yml -i test_inventory.yml --ask-vault-password
```

4. **Verify deployment**:
```bash
# Check service status
ssh your-test-server
sudo systemctl status grafana

# Check service logs
sudo journalctl -u grafana -f

# Check running containers
sudo -u grafana podman ps

# View Grafana logs
sudo -u grafana podman logs grafana
```

5. **Access Grafana**:
- HTTP: `http://your-server-ip:3000`
- HTTPS (with SSL): `https://grafana.example.com`
- Default credentials: `admin` / `changeme` (change immediately!)

### Testing the test_grafana.yml Playbook

The `test_grafana.yml` playbook is designed to test a complete Grafana deployment with all prerequisites on the homester server.

1. **Prerequisites**:
   - Ensure KeePass database has entries for `homester/node` (username, password) and `homester/domain_name`
   - Verify DNS A record points to homester server IP for `dashboard.<domain>`
   - Activate ansible environment: `pyenv shell ansible`

2. **Run the test playbook**:
```bash
export OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES
env no_proxy='*' ansible-playbook playbooks/test_grafana.yml -i hosts.yml --ask-vault-password
```

3. **Verify deployment**:
```bash
# SSH to homester server
ssh homester

# Check Grafana service status
sudo systemctl status grafana

# Check container is running
sudo -u grafana podman ps

# Check NGINX configuration
sudo nginx -t

# Check firewall rules
sudo ufw status

# View Grafana logs
sudo journalctl -u grafana -n 50 --no-pager
```

4. **Access Grafana**:
   - URL: `https://dashboard.<your-domain>`
   - Credentials: Use KeePass `homester/node` credentials
   - First login: You'll be prompted to change the password

5. **Troubleshooting**:
```bash
# Check if all prerequisites are installed
dpkg -l | grep -E "(nginx|podman|ufw)"

# Verify SSL certificate
sudo certbot certificates

# Check NGINX reverse proxy
sudo cat /etc/nginx/sites-available/dashboard.*

# Test NGINX configuration
sudo nginx -t && sudo systemctl reload nginx

# Restart Grafana service if needed
sudo systemctl restart grafana
```

### Troubleshooting Grafana Systemd Service

**Container won't start:**
```bash
# Check service status
sudo systemctl status grafana

# View logs
sudo journalctl -u grafana -n 50 --no-pager

# Check systemd service file
sudo cat /etc/systemd/system/grafana.service
```

**Permission issues:**
```bash
# Check user permissions
sudo -u grafana podman ps

# Verify data directory ownership
ls -la /var/lib/grafana
```

**SELinux issues (if enabled):**
```bash
# Check SELinux context
ls -Z /var/lib/grafana

# Add :Z to volume mounts in service file (already included)
```

**Data persistence:**
```bash
# Backup Grafana data
sudo tar -czf grafana-backup.tar.gz -C /var/lib/grafana .

# Restore Grafana data
sudo tar -xzf grafana-backup.tar.gz -C /var/lib/grafana
```

## Grafana Role Documentation

The `grafana` role installs Grafana using native packages with comprehensive permission checks and security hardening.

### Role Features

- **Native package installation** via official Grafana APT repository
- **Custom grafana.ini template** for configuration management
- **Comprehensive permission checks** before and after deployment
- **SELinux context management** (when enabled)
- **Automatic SSL certificates** via Let's Encrypt
- **Nginx reverse proxy** with WebSocket support for Grafana Live
- **Health monitoring** and permission error detection
- **Security headers** and hardening

### Role Variables

#### Required Variables

```yaml
# Domains for Grafana access
grafana_domains:
  - "dashboard.example.com"
  - "www.dashboard.example.com"

# Grafana admin credentials (override in host_vars with KeePass)
grafana_admin_user: "admin"
grafana_admin_password: "changeme"

# User email for SSL certificates
user:
  email: "admin@example.com"
```

#### Optional Variables

```yaml
# Grafana configuration (grafana.ini format)
grafana_ini_config:
  server:
    http_port: 3000
    root_url: "%(protocol)s://%(domain)s/"
  security:
    admin_user: "{{ grafana_admin_user }}"
    admin_password: "{{ grafana_admin_password }}"
    cookie_secure: true
    cookie_samesite: "lax"
    disable_gravatar: true
  database:
    type: "sqlite3"
    path: "/var/lib/grafana/grafana.db"

# Service management
grafana_service_name: "grafana-server"
grafana_service_state: "started"
grafana_service_enabled: true

# Permission and security
grafana_permission_checks_enabled: true
grafana_auto_fix_permissions: true
grafana_manage_selinux: true
```

### Directory Structure

```
roles/grafana/
├── defaults/
│   └── main.yml           # Configuration variables
├── handlers/
│   └── main.yml           # Service and permission handlers
├── tasks/
│   └── main.yml           # Installation and verification tasks
├── templates/
│   ├── grafana.ini.j2     # Grafana configuration template
│   └── nginx.conf.j2      # Nginx reverse proxy configuration
└── meta/
    └── main.yml           # Role metadata
```

### Usage Example

```yaml
# In host_vars/your_host.yml
grafana_domains:
  - "dashboard.example.com"

grafana_admin_user: "admin"
grafana_admin_password: "{{ vault_grafana_password }}"

user:
  email: "admin@example.com"

# In your playbook
- name: Install Grafana
  hosts: your_servers
  roles:
    - role: grafana
```

### Permission Checks

The role performs comprehensive permission verification:

**Pre-deployment:**
- ✅ Directory ownership verification (grafana:grafana)
- ✅ Permission mode validation (0750)
- ✅ SELinux context configuration (when enabled)
- ✅ Subdirectory creation with proper permissions
- ✅ Systemd service file validation

**Runtime:**
- ✅ Service start verification
- ✅ Health endpoint monitoring
- ✅ Log analysis for permission errors
- ✅ Write access verification

### Migration from grafana_container

To migrate from the container-based installation to native:

1. **Backup existing data** (if needed):
```bash
sudo tar -czf grafana-backup.tar.gz -C /var/lib/grafana .
```

2. **Remove container installation**:
```bash
sudo systemctl stop grafana
sudo systemctl disable grafana
sudo podman rm -f grafana
sudo rm -f /etc/systemd/system/grafana.service
```

3. **Run the native Grafana role**:
```bash
ansible-playbook playbooks/your_playbook.yml -i hosts.yml
```

4. **Verify installation**:
```bash
sudo systemctl status grafana-server
sudo journalctl -u grafana-server -f
curl http://localhost:3000/api/health
```

### Service Management

```bash
# Check service status
sudo systemctl status grafana-server

# View logs
sudo journalctl -u grafana-server -f

# Restart service
sudo systemctl restart grafana-server

# Edit configuration
sudo nano /etc/grafana/grafana.ini

# View configuration
sudo cat /etc/grafana/grafana.ini
```

### Troubleshooting

**Permission issues:**
```bash
# Fix ownership and permissions
sudo chown -R grafana:grafana /var/lib/grafana
sudo chmod -R 0750 /var/lib/grafana

# Fix SELinux context (if enabled)
sudo restorecon -Rv /var/lib/grafana
```

**Service won't start:**
```bash
# Check logs for errors
sudo journalctl -u grafana-server -n 50 --no-pager

# Verify configuration
sudo grafana-cli --config=/etc/grafana/grafana.ini admin reset-admin-password

# Check port availability
sudo ss -tlnp | grep 3000
```

**Health check failures:**
```bash
# Test health endpoint locally
curl http://localhost:3000/api/health

# Test via nginx proxy
curl https://dashboard.example.com/api/health
```

### Differences: Native vs Container Grafana

| Feature | Native Installation | Container (Systemd Service) |
|---------|-------------------|---------------------|
| **Installation Method** | Apt packages | Podman image |
| **Service Scope** | System systemd | System systemd |
| **User Context** | grafana system user | grafana system user (dropped privileges) |
| **Data Storage** | System directories | `/var/lib/grafana` directory |
| **Configuration** | `/etc/grafana/grafana.ini` | Environment variables |
| **Updates** | `apt upgrade` | Image pull and recreate |
| **Isolation** | None | Container namespace |
| **Resource Limits** | System-level | Per-container limits |
| **Service File** | `/lib/systemd/system/grafana-server.service` | `/etc/systemd/system/grafana.service` |

## Grafana Service Role Documentation

The `grafana_service` role provides a self-contained Grafana installation using the official `grafana.grafana` Ansible collection. It follows the same pattern as `headscale_service`, creating a dedicated system user, installing Grafana, setting up SSL certificates, and configuring nginx as a reverse proxy.

### Role Features

- **Self-contained installation** - Creates dedicated `grafana` system user
- **Uses grafana.grafana collection** - Leverages official community Ansible collection
- **Custom grafana.ini template** - Template-based configuration management
- **SSL certificates** - Automatic Let's Encrypt certificate generation via certbot role
- **Nginx reverse proxy** - Secure HTTPS access with WebSocket support
- **Systemd service** - Automatic service enablement and management
- **Basic authentication** - Native Grafana authentication (admin credentials from KeePass)

### Role Structure

```
roles/grafana_service/
├── defaults/
│   └── main.yml           # Role variables
├── handlers/
│   └── main.yml           # Service restart handlers
├── tasks/
│   ├── main.yml           # Orchestrator
│   ├── install_grafana.yml # Install Grafana using collection
│   └── configure_grafana.yml # Configure Grafana directories and config
└── templates/
    ├── grafana.ini.j2     # Grafana configuration template
    └── nginx.conf.j2      # Nginx reverse proxy configuration
```

### Role Variables

#### Required Variables

```yaml
# Domains for Grafana access
grafana_domains:
  - "dashboard.example.com"
  - "www.dashboard.example.com"

# Primary domain for SSL certificate
grafana_primary_domain: "dashboard.example.com"

# Grafana admin credentials (from KeePass in production)
grafana_admin_user: "admin"
grafana_admin_password: "changeme"

# User email for Let's Encrypt
user:
  email: "admin@example.com"
```

#### Optional Variables

```yaml
# Grafana version (empty for latest)
grafana_version: ""

# Service configuration
grafana_user: grafana
grafana_group: grafana
grafana_service_name: grafana-server
grafana_port: 3000

# Directories
grafana_config_dir: /etc/grafana
grafana_data_dir: /var/lib/grafana
grafana_log_dir: /var/log/grafana
grafana_plugins_dir: /var/lib/grafana/plugins
grafana_provisioning_dir: /etc/grafana/provisioning

# Server configuration (grafana.ini [server] section)
grafana_server_section:
  http_addr: "127.0.0.1"
  http_port: 3000
  domain: "dashboard.example.com"

# Security configuration (grafana.ini [security] section)
grafana_security_section:
  admin_user: "admin"
  admin_password: "changeme"
```

### Usage Example

In `host_vars/homester.yaml`:

```yaml
# From KeePass
grafana_admin_user: "{{ lookup('keepass', 'homester/user', 'username') }}"
grafana_admin_password: "{{ lookup('keepass', 'homester/user', 'password') }}"

# Domains
grafana_domains:
  - "dashboard.{{ lookup('keepass', 'homester/domain_name', 'username') }}"
  - "www.dashboard.{{ lookup('keepass', 'homester/domain_name', 'username') }}"
grafana_primary_domain: "dashboard.{{ lookup('keepass', 'homester/domain_name', 'username') }}"

# User email
user:
  email: "{{ lookup('keepass', 'homester/email', 'username') }}"
```

In your playbook (e.g., `playbooks/homester.yml`):

```yaml
- name: 'Install Grafana'
  hosts: vpn
  become: true
  remote_user: "{{ user.name }}"
  vars:
    ansible_ssh_private_key_file: "{{ user.ssh_key_path }}/{{ user.ssh_key_file }}"
  tasks:
    - name: 'Import grafana service role'
      ansible.builtin.import_role:
        name: grafana_service
```

### Testing the grafana_service Role

The `homester.yml` playbook uses the `grafana_service` role for complete Grafana setup.

1. **Prerequisites**:
   - Ensure KeePass database has entries for `homester/user` and `homester/domain_name`
   - Verify DNS A record points to homester server IP for `dashboard.<domain>`
   - Activate ansible environment: `pyenv shell ansible`

2. **Run the homester playbook**:
```bash
export OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES
env no_proxy='*' ansible-playbook playbooks/homester.yml -i hosts.yml --ask-vault-password
```

3. **Verify deployment**:
```bash
# SSH to homester server
ssh homester

# Check Grafana service status
sudo systemctl status grafana-server

# Check Grafana is listening
sudo ss -tlnp | grep 3000

# Check NGINX configuration
sudo nginx -t

# Check SSL certificate
sudo certbot certificates

# View Grafana logs
sudo journalctl -u grafana-server -n 50 --no-pager
```

4. **Access Grafana**:
   - URL: `https://dashboard.<your-domain>`
   - Credentials: Use KeePass `homester/user` credentials
   - First login: You'll be prompted to change the password

### Installation Steps

The `grafana_service` role performs these steps in order:

1. **Create system user** - Creates `grafana` system user with no login shell
2. **Install Grafana** - Uses `grafana.grafana.grafana` collection to:
   - Add official Grafana APT repository
   - Import GPG key
   - Install Grafana package
   - Create and enable systemd service (package default)
   - Start the service
3. **Configure directories** - Ensures all directories exist with proper permissions (grafana:grafana)
4. **Deploy configuration** - Installs custom `grafana.ini` from template with admin credentials (triggers service restart)
5. **Setup SSL** - Uses `certbot` role to obtain Let's Encrypt certificate
6. **Configure nginx** - Uses `nginx_config` role to deploy reverse proxy configuration

### Service Management

```bash
# Check service status
sudo systemctl status grafana-server

# View logs
sudo journalctl -u grafana-server -f

# Restart service
sudo systemctl restart grafana-server

# Edit configuration
sudo nano /etc/grafana/grafana.ini

# View configuration
sudo cat /etc/grafana/grafana.ini

# Check Grafana version
grafana-server -v
```

### Troubleshooting

**Service won't start:**
```bash
# Check service status and logs
sudo systemctl status grafana-server
sudo journalctl -u grafana-server -n 50 --no-pager

# Verify configuration
sudo grafana-server --config=/etc/grafana/grafana.ini --homepath=/usr/share/grafana

# Check port availability
sudo ss -tlnp | grep 3000
```

**Permission issues:**
```bash
# Fix ownership and permissions
sudo chown -R grafana:grafana /var/lib/grafana
sudo chown -R grafana:grafana /var/log/grafana
sudo chown -R grafana:grafana /etc/grafana
sudo chmod 0640 /etc/grafana/grafana.ini
```

**SSL certificate issues:**
```bash
# Check certificate status
sudo certbot certificates

# Renew certificate manually
sudo certbot renew

# Check nginx configuration
sudo nginx -t
sudo systemctl reload nginx
```

**Health check:**
```bash
# Test health endpoint locally
curl http://localhost:3000/api/health

# Test via nginx proxy
curl https://dashboard.example.com/api/health
```

### Migration from grafana or grafana_container

To migrate from the previous `grafana` or `grafana_container` roles to `grafana_service`:

1. **Backup existing data** (if needed):
```bash
# For container installation
sudo podman exec grafana grafana-cli admin reset-admin-password newpass
sudo tar -czf grafana-backup.tar.gz -C /var/lib/grafana .

# For native installation
sudo tar -czf grafana-backup.tar.gz -C /var/lib/grafana .
```

2. **Stop and remove old installation**:
```bash
# Stop old service
sudo systemctl stop grafana  # or grafana-server
sudo systemctl disable grafana

# For container installation
sudo podman rm -f grafana
sudo rm -f /etc/systemd/system/grafana.service

# For native installation
sudo apt remove grafana
```

3. **Run the new grafana_service role**:
```bash
ansible-playbook playbooks/homester.yml -i hosts.yml
```

4. **Verify installation**:
```bash
sudo systemctl status grafana-server
sudo journalctl -u grafana-server -f
curl https://dashboard.example.com
```

### Key Differences from Other Roles

| Feature | grafana_service | grafana (old) | grafana_container | headscale_service |
|---------|----------------|---------------|-------------------|-------------------|
| **Installation Method** | grafana.grafana collection | Manual APT setup | Podman container | Binary download |
| **System User** | Creates grafana user + package user | Uses package user | Creates grafana user | Creates headscale user |
| **Configuration** | Template-based | Template-based | Environment variables | Template-based |
| **SSL/NGINX** | certbot + nginx_config roles | Inline in role | Inline in role | certbot + nginx_config roles |
| **Service Type** | System systemd (package) | System systemd | System systemd (container) | Custom systemd service |
| **Service File** | Package default | Package default | Custom template | Custom template |
| **Pattern** | Uses official collection | Custom | Custom | Custom binary install |

The `grafana_service` role is the recommended approach for Grafana installation because it:
- Leverages the official `grafana.grafana` Ansible collection
- Uses the package's systemd service file (follows official docs)
- Integrates with certbot and nginx_config helper roles
- Requires minimal custom configuration (only grafana.ini)
## Misc tips

### Enable tailscale custom coordinator on GliNet router

patch /usr/bin/gl_tailscale

```
timeout 10 /usr/sbin/tailscale up --login-server https://headscale.example.com --reset --accept-routes $param --timeout 3s --accept-dns=false > /dev/null
```
