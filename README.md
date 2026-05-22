# Ansible Repository for Server Management

## Environment Setup from Scratch

```bash
# 1. Install pyenv and virtualenv plugin (macOS)
brew install pyenv
brew install pyenv-virtualenv

# 2. Install Python 3.11.5
pyenv install 3.11.5

# 3. Create ansible virtualenv
pyenv virtualenv 3.11.5 ansible

# 4. Activate the ansible environment
pyenv shell ansible

# 5. Upgrade pip
pip install --upgrade pip

# 6. Install Python dependencies
pip install -r requirements.txt

# 7. Install Ansible collections and roles
ansible-galaxy collection install -r requirements.yml
ansible-galaxy role install -r requirements.yml

# 8. Install GNU tar for macOS compatibility
brew install gnu-tar

# 9. Setup localhost KeePass plugin
mkdir -p ~/.ansible/plugins/lookup
cp ~/.ansible/collections/ansible_collections/viczem/keepass/plugins/lookup/keepass.py ~/.ansible/plugins/lookup/

# 10. Verify installation
ansible --version
ansible-galaxy collection list
```

## Running Playbooks

```bash
# Activate environment
pyenv shell ansible

# Run playbook
export OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES
env no_proxy='*' ansible-playbook playbooks/example.yml -i hosts.yml --ask-vault-password
```

## Available Playbooks

- **playbooks/example.yml** - Leaf node setup with Node Exporter (TLS enabled)
- **playbooks/headplane.yml** - Headplane server with Headscale, Prometheus, and monitoring stack

## Tailscale Custom Coordinator on GliNet Router

Enable Tailscale custom coordinator:

```bash
# Patch /usr/bin/gl_tailscale
timeout 10 /usr/sbin/tailscale up --login-server https://headscale.example.com --reset --accept-routes $param --timeout 3s --accept-dns=false > /dev/null
```
