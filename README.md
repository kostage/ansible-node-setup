# Ansible Repository for Server Management

This repository uses a modular role architecture for managing web services including Grafana and Headscale. The architecture eliminates code duplication and provides consistent configuration patterns across all services.

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
- **`grafana.grafana.grafana`** - Grafana monitoring dashboards (community role, native installation)
- **`grafana_container`** - Grafana monitoring dashboards using Podman with systemd system service

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
## Misc tips

### Enable tailscale custom coordinator on GliNet router

patch /usr/bin/gl_tailscale

```
timeout 10 /usr/sbin/tailscale up --login-server https://headscale.example.com --reset --accept-routes $param --timeout 3s --accept-dns=false > /dev/null
```
