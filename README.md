# Ansible Repository for Server Management

This repository uses a modular role architecture for managing web services including Grafana, Headscale, and Headplane. The architecture eliminates code duplication and provides consistent configuration patterns across all services.

## Quick Start

### Python Virtual Environment Setup

```bash
brew install pyenv
brew install pyenv-virtualenv
pyenv install 3.11.5
pyenv virtualenv 3.11.5 ansible
pyenv shell ansible
```

### Install Ansible Requirements

```bash
ansible-galaxy install -r requirements.yml
```

### Localhost Setup

```bash
brew install gnu-tar
cp ~/.ansible/collections/ansible_collections/viczem/keepass/plugins/lookup/keepass.py ~/.ansible/plugins/lookup
```

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
- **`headplane_service`** - Headplane web UI for Headscale management
- **`grafana.grafana.grafana`** - Grafana monitoring dashboards (community role)

### Flat Variable Structure

All services use direct variables with no nested structures:

```yaml
# Service domains (direct variables)
headscale_domains:
  - "headscale.example.com"
  - "www.headscale.example.com"

headplane_domains:
  - "headplane.example.com"

# OAuth2 configuration (flat)
headplane_oauth2_port: 4190
headplane_oauth2_client_id: "oauth_client_id"
headplane_oauth2_client_secret: "oauth_client_secret"

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
- **Purpose**: Headscale + Headplane + Grafana on single server
- **Services**: Headscale, Headplane, Grafana
- **Architecture**: Completely flat structure with explicit steps
- **Features**: Every setup step visible, no abstraction layers

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
5. OAuth2 authentication flow

### Example Testing Commands

```bash
# Check service status
systemctl status headscale
systemctl status headplane
systemctl status oauth2_proxy_grafana
systemctl status nginx

# Test SSL certificates
certbot certificates

# Verify NGINX configuration
nginx -t

# Check port accessibility
netstat -tlnp | grep -E ':(80|443|3000|4444|7777|4180|4190)'

# Test service endpoints
curl -k https://dashboard.example.com
curl -k https://headscale.example.com
curl -k https://headplane.example.com
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
1. **User and基础 setup** - Create user, install packages, Docker
2. **NGINX Foundation** - Install NGINX (community role)
3. **Service Installation** - Individual service roles (Headscale, Headplane, Grafana)
4. **SSL Certificates** - Separate certbot calls for each service
5. **OAuth2 Proxies** - Individual OAuth2 proxy setup for each service
6. **NGINX Configuration** - Separate NGINX config for each service
7. **Security Hardening** - SSH and firewall configuration

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
│   ├── headplane.yml         # New flat architecture playbook
│   └── grafana.yml           # Legacy monitoring playbook
└── roles/                    # Custom and community roles
    ├── nginx_config/         # Universal NGINX configuration
    ├── certbot/             # SSL certificate management
    ├── oauth2_proxy/        # OAuth2 proxy management
    ├── headscale_service/   # Headscale-specific setup
    ├── headplane_service/   # Headplane-specific setup
    └── [existing custom roles]
```

## Troubleshooting

### Common Issues

1. **Certificate Issues**
   - Check domain DNS resolution
   - Verify port 80 accessibility for certbot challenges
   - Check certbot logs: `journalctl -u certbot`

2. **OAuth2 Issues**
   - Verify OAuth2 service status
   - Check configuration file syntax
   - Validate Google OAuth2 credentials

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

### Misc
```
ssh-add -D
ssh-keygen -R 70.34.250.23
ssh-copy-id -i ~/Doroshenko/nodes/vultrssh.pub root@70.34.250.23
ssh-add ~/Doroshenko/nodes/vultrssh
```
