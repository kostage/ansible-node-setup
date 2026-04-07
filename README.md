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

## Configuration Management

### Secrets Management
All secrets are stored in KeePass database using the `viczem.keepass` collection:
- No secrets in repository
- Centralized secret management
- Environment-specific configurations

### Variable Structure
- `services[]` - Service configurations using new architecture
- Service-specific vars (e.g., `grafana_config`) for detailed settings
- Legacy variables maintained for backward compatibility

## Testing Procedures

### Pre-flight Checks
1. Verify KeePass database accessibility
2. Check host variable configurations
3. Validate firewall rules
4. Test SSL certificate domains

### Service Testing
For each service, verify:
1. Service is running: `systemctl status <service>`
2. Port accessibility: `netstat -tlnp | grep <port>`
3. SSL certificates: `certbot certificates`
4. NGINX configuration: `nginx -t`
5. Grafana basic authentication (login with non-root user credentials)

### Example Testing Commands

```bash
# Check service status
systemctl status headscale
systemctl status grafana-server
systemctl status prometheus
systemctl status alertmanager
systemctl status nginx

# Test SSL certificates
certbot certificates

# Verify NGINX configuration
nginx -t

# Check port accessibility
netstat -tlnp | grep -E ':(80|443|3000|4444|9090)'

# Test service endpoints
curl -k https://dashboard.example.com
curl -k https://headscale.example.com
```

## Migration Strategy

### Phase 1: New Services
- All new services use the modular architecture
- Documented in `ROLES_ARCHITECTURE.md`

### Phase 2: Migration
- Existing services gradually migrated to new architecture
- Backward compatibility maintained during transition
- Example configurations provided

### Phase 3: Cleanup
- Remove duplicate code
- Standardize all services under new architecture
- Update documentation

## Role Details

### Playbook Structure
The `headplane.yml` playbook contains completely explicit steps:
1. **User and基础 setup** - Create user, install packages
2. **NGINX Foundation** - Install NGINX (community role)
3. **Service Installation** - Individual service roles (Headscale, Grafana, Prometheus, Alertmanager, Node Exporter)
4. **Security Hardening** - SSH and firewall configuration

### nginx_config Role
Features universal template that handles:
- HTTP to HTTPS redirects
- SSL certificate integration
- Conditional OAuth2 authentication
- WebSocket support
- Security headers
- Multi-domain support

### certbot Role
Handles SSL certificate management:
- Initial HTTP-only configuration
- Certificate generation
- Automatic renewal setup
- Multi-domain support
- Certificate cleanup

### oauth2_proxy Role
Manages OAuth2 authentication:
- Multiple proxy instances
- Template-based configuration
- Systemd service management
- Cookie and session management

## File Structure

```
├── ROLES_ARCHITECTURE.md     # Detailed architecture documentation
├── requirements.txt          # Python dependencies
├── requirements.yml          # Ansible collections and roles
├── hosts.yml                 # Inventory configuration
├── host_vars/                # Host-specific variables
├── group_vars/               # Group-specific variables
├── playbooks/                # Main playbook files
│   ├── headplane.yml         # Headscale + Grafana + monitoring stack
│   └── grafana.yml           # Legacy monitoring playbook
└── roles/                    # Custom and community roles
    ├── headscale_service/    # Headscale-specific setup
    ├── grafana/              # Grafana service with NGINX config
    ├── prometheus/           # Prometheus monitoring
    ├── alertmanager/         # Alert management
    └── [existing custom roles]
```

## Troubleshooting

### Common Issues

1. **Certificate Issues**
   - Check domain DNS resolution
   - Verify port 80 accessibility for certbot challenges
   - Check certbot logs: `journalctl -u certbot`

2. **Authentication Issues**
   - For Grafana: Verify admin credentials from KeePass
   - Check Grafana service logs: `journalctl -u grafana-server`
   - Ensure NGINX is properly proxying to Grafana

3. **NGINX Issues**
   - Test configuration: `nginx -t`
   - Check logs: `journalctl -u nginx`
   - Verify SSL certificate paths

4. **Service Issues**
   - Check systemd logs: `journalctl -u <service>`
   - Verify configuration files
   - Check port conflicts

### Getting Help

1. Check service status: `systemctl status <service>`
2. Review logs: `journalctl -u <service> -f`
3. Test configurations: `nginx -t`, `certbot certificates`
4. Check variable values: `ansible-playbook --check` mode

### SSH Debugging Commands

When troubleshooting package installation issues, connect to the server and check:

```bash
# Connect to server
ssh -p <ssh_port> <user>@<server_ip>

# Check apt repository configuration
ls -la /etc/apt/sources.list.d/
cat /etc/apt/sources.list.d/grafana.list

# Check GPG keys
ls -la /usr/share/keyrings/grafana.key

# Check if package is available
apt-cache policy grafana
apt-cache search grafana

# Update apt cache manually
sudo apt update

# Check apt logs
sudo tail -n 50 /var/log/apt/history.log
sudo tail -n 50 /var/log/apt/term.log

# Manually add repository if needed (temporary fix)
echo "deb [signed-by=/usr/share/keyrings/grafana.key] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update
apt-cache policy grafana

# Fix corrupted apt database
sudo mkdir -p /var/lib/apt/lists
sudo apt-get update
```
