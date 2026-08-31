## The project structure is:
```
~/gitlab-ansible/
├── ansible.cfg
├── requirements.yml
├── inventory/
│   └── hosts.yml
├── group_vars/
│   ├── gitlab.yml              # single central configuration file
│   └── gitlab_vault.yml       # encrypted secrets only
├── playbooks/
│   ├── site.yml
│   ├── prepare.yml
│   ├── deploy.yml
│   ├── harden.yml
│   └── backup.yml
└── roles/
    ├── host_prepare/
    ├── docker/
    ├── firewall/
    ├── gitlab/
    └── gitlab_backup/
```
## Encrypt it immediately
### After configuring `./group_vars/gitlab.yaml` and `./group_vars/gitlab_vault.yml` 
### From the project root:
```
cd ~/gitlab-ansible

ansible-vault encrypt group_vars/gitlab_vault.yml
```
## Install/update the collections:
```
cd ~/gitlab-ansible
ansible-galaxy collection install -r requirements.yml
```