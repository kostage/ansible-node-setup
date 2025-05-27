### python virtualenv

```bash
brew install pyenv
brew install pyenv-virtualenv
pyenv install 3.11.5
pyenv virtualenv 3.11.5 ansible
pyenv shell ansible
```

### ansible requirements
```bash
ansible-galaxy install -r requirements.yml
```

### localhost
```bash
brew install gnu-tar
cp ~/.ansible/collections/ansible_collections/viczem/keepass/plugins/lookup/keepass.py ~/.ansible/plugins/lookup

```

```bash
export OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES
env no_proxy='*' ansible-playbook --ask-vault-password playbooks/playbook.yml -i hosts.yml
```