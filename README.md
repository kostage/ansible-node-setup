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
- **`grafana.grafana.grafana`** - Grafana monitoring dashboards (community role)

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
