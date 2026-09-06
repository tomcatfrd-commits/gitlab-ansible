# GitLab Ansible Deployment

Ansible-based deployment and hardening framework for running **GitLab Self-Managed EE** in Docker on a dedicated Ubuntu server.

The project is designed around a controlled, reproducible deployment model with:

* Ubuntu 26.04 LTS
* x86_64 / amd64
* Docker Engine and Docker Compose v2
* GitLab EE 19.3.1
* Immutable GitLab image digest pinning
* Self-signed TLS
* UFW host firewall
* Dedicated Docker `DOCKER-USER` firewall policy
* IPv6 disabled
* GitLab security hardening
* Persistent GitLab configuration, data, logs, and backup directories
* Optional systemd-based GitLab backups
* Ansible collections pinned to known versions

The project intentionally separates host preparation, Docker configuration, firewall configuration, GitLab deployment, GitLab hardening, and backup management into independent Ansible roles.

---

## 1. Project Goals

The primary goal is to provide a repeatable and auditable way to deploy a hardened GitLab instance without manually configuring the server.

The deployment should provide the following properties:

1. Reproducibility

   The same Ansible configuration should produce an equivalent GitLab deployment on another supported server.

2. Configuration as Code

   Host, Docker, firewall, TLS, GitLab, and backup configuration are maintained in Ansible variables and templates rather than manually configured on the server.

3. Image Integrity

   The GitLab container image is pinned to a specific digest instead of relying only on a mutable image tag.

4. Network Restriction

   Only explicitly required inbound ports are exposed.

5. Container Isolation

   GitLab does not run in privileged mode and does not receive access to the Docker socket.

6. Host Hardening

   The deployment applies selected host-level security controls, including disabling IPv6 and configuring required kernel parameters.

7. Operational Separation

   Deployment, hardening, and backup operations are separated into dedicated playbooks and roles.

---

# 2. Architecture

The deployment uses the following architecture:

```text
                         Administrator
                              |
                              | SSH : 2222
                              |
                              v
                    +--------------------+
                    | Ubuntu 26.04       |
                    | GitLab Server      |
                    |                    |
                    | UFW                |
                    | Docker Engine      |
                    +---------+----------+
                              |
                              | Docker bridge
                              |
                    +---------v----------+
                    | GitLab Container   |
                    |                    |
                    | GitLab EE          |
                    | 19.3.1-ee.0       |
                    +---------+----------+
                              |
                +-------------+-------------+
                |             |             |
                v             v             v
             config         logs          data
                |             |             |
                +-------------+-------------+
                              |
                         backups
```

External network access is intentionally limited to:

```text
TCP 2222  -> Administrative SSH
TCP 443   -> GitLab HTTPS
TCP 22    -> GitLab SSH
```

GitLab's internal HTTP listener uses port 80 inside the container but is not published directly to the host.

---

# 3. Deployment Model

The project uses four primary stages:

```text
prepare
   |
   v
deploy
   |
   v
harden
   |
   v
backup
```

These stages are orchestrated by:

```text
playbooks/site.yml
```

The complete flow is:

```text
site.yml
│
├── prepare.yml
│   ├── host_prepare
│   ├── docker
│   └── firewall
│
├── deploy.yml
│   └── gitlab
│
├── harden.yml
│   └── gitlab hardening
│
└── backup.yml
    └── gitlab_backup (optional)
```

This separation allows individual stages to be executed independently when required.

---

# 4. Supported Environment

The current deployment is intentionally restricted to:

| Component          | Requirement       |
| ------------------ | ----------------- |
| Operating System   | Ubuntu 26.04      |
| Architecture       | x86_64 / amd64    |
| CPU                | Minimum 8 vCPU    |
| RAM                | Minimum 16 GB     |
| Administrative SSH | TCP 2222          |
| GitLab HTTPS       | TCP 443           |
| GitLab SSH         | TCP 22            |
| Container Runtime  | Docker Engine     |
| Compose            | Docker Compose v2 |
| GitLab             | EE 19.3.1-ee.0    |
| Network Mode       | Docker bridge     |
| IPv6               | Disabled          |

The Ansible validation tasks explicitly verify the operating system, architecture, CPU, and memory requirements.

---

# 5. Repository Structure

```text
gitlab-ansible/
│
├── ansible.cfg
├── requirements.yml
│
├── inventory/
│   └── hosts.yml
│
├── group_vars/
│   ├── gitlab.yml
│   └── gitlab_vault.yml
│
├── playbooks/
│   ├── site.yml
│   ├── prepare.yml
│   ├── deploy.yml
│   ├── harden.yml
│   └── backup.yml
│
└── roles/
    │
    ├── host_prepare/
    │   ├── defaults/
    │   ├── tasks/
    │   ├── handlers/
    │   └── templates/
    │
    ├── docker/
    │   ├── defaults/
    │   ├── tasks/
    │   ├── handlers/
    │   └── templates/
    │
    ├── firewall/
    │   ├── defaults/
    │   ├── tasks/
    │   ├── handlers/
    │   └── templates/
    │
    ├── gitlab/
    │   ├── defaults/
    │   ├── tasks/
    │   ├── handlers/
    │   └── templates/
    │
    └── gitlab_backup/
        ├── defaults/
        ├── tasks/
        ├── handlers/
        └── templates/
```

---

# 6. Ansible Requirements

The project uses the following Ansible collections:

```yaml
collections:
  - name: community.docker
    version: "4.8.1"

  - name: ansible.posix
    version: "2.1.0"

  - name: community.general
    version: "11.4.0"

  - name: community.crypto
    version: "3.3.0"
```

Install them with:

```bash
ansible-galaxy collection install -r requirements.yml
```

Verify the installation:

```bash
ansible-galaxy collection list
```

---

# 7. Ansible Control Node

The target server is Ubuntu 26.04, but the Ansible control node can be a separate Linux system.

For Windows administrators, WSL2 with Ubuntu is recommended rather than attempting to operate this project from a native Windows Ansible environment.

Example:

```text
Windows
│
├── D:\Deployment\gitlab-ansible
│
└── WSL2
    └── Ubuntu
        ├── Python
        ├── Ansible
        └── Git
```

The repository can remain on the Windows filesystem and be accessed from WSL through `/mnt/d`.

Example:

```bash
cd /mnt/d/Deployment/gitlab-ansible
```

---

# 8. Inventory

The inventory is located at:

```text
inventory/hosts.yml
```

Current structure:

```yaml
all:
  children:
    gitlab:
      hosts:
        gitlab-server:
```

The actual connection parameters are intentionally maintained in:

```text
group_vars/gitlab.yml
```

---

# 9. Required Configuration

Before deployment, update the `CHANGE_ME` values in:

```text
group_vars/gitlab.yml
```

At minimum:

```yaml
ansible_host: "CHANGE_ME"
ansible_user: "CHANGE_ME"
gitlab_fqdn: "CHANGE_ME"
```

Example:

```yaml
ansible_host: "192.0.2.10"
ansible_user: "ansibleadmin"
gitlab_fqdn: "gitlab.example.internal"
```

Do not use the example values literally.

The administrative SSH port is:

```yaml
ansible_port: 2222
```

The GitLab SSH port is:

```yaml
gitlab_ssh_port: 22
```

These ports must remain different.

---

# 10. GitLab Image Pinning

The deployment uses:

```yaml
gitlab_image_repository: "gitlab/gitlab-ee"
gitlab_image_version: "19.3.1-ee.0"
```

The image is also pinned to a digest:

```yaml
gitlab_image_digest: "sha256:f204f55c2825659fd2b4ae6a6acdcd79a274e3c1dfc98b74a6f0c7b5f9da14e6e6"
```

The resulting image reference is conceptually:

```text
gitlab/gitlab-ee:19.3.1-ee.0@sha256:f204f55c2825659fd2b4ae6a6acdcd79a274e3c1dfc98b74a6f0c7b5f9da14e6e6
```

The deployment validates the image digest before starting GitLab.

This prevents a deployment from silently using a different image while retaining the same tag.

When upgrading GitLab, the version and digest should be deliberately reviewed and changed together.

---

# 11. Docker

Docker is installed from the Ubuntu repositories.

The role first removes conflicting Docker packages and then installs the configured Ubuntu Docker packages.

The project deliberately does not add the Ansible/deployment user to the `docker` group.

This is intentional because membership in the Docker group can provide effectively root-equivalent control over the host.

Docker administration is performed using Ansible privilege escalation.

The Docker daemon is configured with:

* JSON file logging
* log rotation
* live restore
* userland proxy disabled
* iptables integration enabled
* IPv6 Docker filtering enabled by default at the Docker-role level

However, IPv6 is disabled at the host level for this deployment, so IPv6 networking is not used.

---

# 12. Docker Container Security

GitLab is deployed using Docker Compose.

The container is configured with:

```text
privileged: false
no-new-privileges: true
network_mode: bridge
```

The Docker socket is not mounted into the GitLab container.

The deployment therefore does not use:

```text
/var/run/docker.sock
```

inside the GitLab container.

Optional Linux capability dropping is configurable, but the default is intentionally empty because GitLab Omnibus may require capabilities that should not be removed without validating the resulting installation.

---

# 13. GitLab Container Ports

Only the following container ports are published:

```text
Host TCP 443 -> Container TCP 443
Host TCP 22  -> Container TCP 22
```

The container's internal HTTP port 80 is not published directly.

The administrative SSH port belongs to the host:

```text
Host TCP 2222 -> Ubuntu SSH service
```

Therefore:

```text
2222 = server administration
22   = GitLab SSH
443  = GitLab HTTPS
```

---

# 14. TLS

TLS is generated locally by Ansible.

The current configuration uses:

```text
RSA private key
4096-bit key
SHA-256 certificate signature
Self-signed certificate
825-day validity
FQDN as Common Name
FQDN as Subject Alternative Name
```

The private key is stored on the host with restrictive permissions.

The TLS directory is protected as:

```text
root:root
0700
```

The private key is:

```text
root:root
0600
```

The certificate is configured for server authentication.

Because the certificate is self-signed, clients must explicitly trust the certificate or its issuing trust anchor.

For production environments with an internal PKI, the TLS implementation can be adapted to use an organizational CA.

---

# 15. GitLab HTTPS Configuration

GitLab's external URL is:

```text
https://{{ gitlab_fqdn }}
```

The Omnibus NGINX configuration enables HTTPS and disables automatic Let's Encrypt management.

The deployment enables:

```text
TLS 1.2
TLS 1.3
HSTS
Secure session cookies
Hidden NGINX server tokens
```

The HSTS configuration uses:

```text
max-age = 31536000
includeSubdomains = true
preload = false
```

HTTP-to-HTTPS redirection is enabled.

---

# 16. GitLab Security Hardening

The hardening stage applies GitLab application-level security settings.

The current policy includes:

### Registration

New-user signup is disabled.

```yaml
gitlab_signup_enabled: false
```

### Admin mode

Admin mode is enabled.

```yaml
gitlab_admin_mode_enabled: true
```

### Two-Factor Authentication

Mandatory 2FA is enabled.

The configured grace period is:

```text
8 hours
```

The hardening task verifies both the requirement and grace period.

Admin 2FA is also enforced according to the configured policy.

### Unknown Sign-In Notifications

Notifications for unknown sign-ins are enabled.

### Personal Access Tokens

PAT expiration is required according to the configured GitLab policy.

---

# 17. IPv6

IPv6 is intentionally disabled for this deployment.

The central policy is:

```yaml
gitlab_disable_ipv6: true
```

Host-level persistent sysctl settings disable IPv6:

```text
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```

UFW is configured with:

```text
IPV6=no
```

This design intentionally avoids maintaining a separate IPv6 firewall policy.

The deployment therefore uses IPv4 networking only.

---

# 18. UFW Firewall

UFW is configured with a deny-by-default inbound policy.

The default policies are centrally configurable.

The deployment explicitly allows the required inbound TCP ports.

For the current configuration these are:

```text
2222/tcp
443/tcp
22/tcp
```

The administrative SSH rule is created before UFW is enabled.

This is important because enabling UFW before permitting the management SSH port could disconnect the administrator.

After configuration, the role verifies that UFW is active.

---

# 19. Docker Firewall

Docker manages its own packet-filtering rules for published ports.

Therefore, UFW alone is not treated as the complete security boundary for Docker-published services.

The project creates a dedicated Docker firewall chain:

```text
GITLAB-DOCKER-USER
```

The chain is connected through:

```text
DOCKER-USER
```

The policy is conceptually:

```text
DOCKER-USER
    |
    v
GITLAB-DOCKER-USER
    |
    +-- ESTABLISHED,RELATED -> ACCEPT
    |
    +-- GitLab published ports -> ACCEPT
    |
    +-- container-originated traffic -> RETURN
    |
    +-- other external traffic -> DROP
```

The Docker firewall is implemented using a dedicated systemd service.

The service can recreate the policy when Docker is restarted.

The design avoids flushing unrelated rules from `DOCKER-USER`.

---

# 20. Persistent GitLab Storage

GitLab data is stored outside the container using host bind mounts.

The deployment maintains separate directories for:

```text
config/
logs/
data/
backups/
```

The default root location is under the deployment user's home directory:

```text
/home/<deployment-user>/gitlab/
```

The resulting structure is approximately:

```text
/home/<deployment-user>/gitlab/
├── compose/
├── config/
├── logs/
├── data/
└── backups/
```

GitLab configuration and application data therefore survive container recreation.

---

# 21. GitLab Backup

Backups are currently **disabled by default**.

The central configuration is:

```yaml
gitlab_backup_enabled: false
```

When enabled, the backup role creates:

```text
systemd service
systemd timer
```

The backup service invokes GitLab's native backup mechanism inside the GitLab container.

The timer supports:

```text
Persistent=true
```

so a missed scheduled execution can be triggered when the system becomes available again.

The default schedule is:

```text
02:00
```

with a randomized delay.

The configured default retention is:

```text
14 days
```

Enabling the backup feature does not by itself constitute a tested disaster-recovery strategy. A production deployment should include periodic restore testing.

---

# 22. Playbooks

## `site.yml`

Runs the complete deployment workflow:

```text
prepare
deploy
harden
backup
```

Use:

```bash
ansible-playbook playbooks/site.yml
```

---

## `prepare.yml`

Prepares the server.

It validates:

* Ubuntu version
* CPU architecture
* CPU count
* memory
* deployment user
* required host packages

It also configures:

* system time synchronization
* required sysctl settings
* IPv6 policy
* GitLab directories
* Docker
* UFW

Run:

```bash
ansible-playbook playbooks/prepare.yml
```

---

## `deploy.yml`

Deploys GitLab.

It validates the required GitLab configuration and then performs:

```text
TLS preparation
GitLab configuration
Docker Compose configuration
image validation
container deployment
handler execution
health validation
```

Run:

```bash
ansible-playbook playbooks/deploy.yml
```

---

## `harden.yml`

Applies GitLab application-level security settings.

Run:

```bash
ansible-playbook playbooks/harden.yml
```

---

## `backup.yml`

Manages the optional GitLab backup configuration.

When:

```yaml
gitlab_backup_enabled: false
```

the backup role is skipped.

Run:

```bash
ansible-playbook playbooks/backup.yml
```

---

# 23. Recommended Validation Workflow

Do not immediately run the complete deployment on a production server.

First validate the Ansible project.

### 23.1 Check repository state

```bash
git status
```

The working tree should be clean before deployment.

### 23.2 Check for whitespace errors

```bash
git diff --check
```

### 23.3 Install collections

```bash
ansible-galaxy collection install -r requirements.yml
```

### 23.4 Syntax validation

```bash
ansible-playbook --syntax-check playbooks/site.yml
```

### 23.5 Lint

```bash
ansible-lint
```

If the project does not yet include an `.ansible-lint` configuration, use the default Ansible Lint rules initially and address actual findings before deployment.

---

# 24. SSH Connectivity Test

Before running Ansible, verify normal SSH access to the target server.

Example:

```bash
ssh -p 2222 ansibleadmin@192.0.2.10
```

Replace the example values with the actual configured host and user.

Then test Ansible connectivity:

```bash
ansible gitlab -m ansible.builtin.ping
```

Expected result:

```text
gitlab-server | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

---

# 25. Ansible Check Mode

Where supported, use check mode before making changes:

```bash
ansible-playbook playbooks/site.yml --check
```

Check mode is useful but should not be considered a complete substitute for a real deployment.

Some operations, particularly Docker state changes and service interactions, may not fully represent their runtime behavior in check mode.

---

# 26. Deployment Sequence

The recommended deployment sequence is:

```text
1. Configure group_vars
2. Verify inventory
3. Install Ansible collections
4. Run syntax check
5. Run ansible-lint
6. Verify SSH connectivity
7. Run prepare.yml
8. Validate host
9. Run deploy.yml
10. Validate GitLab
11. Run harden.yml
12. Validate security settings
13. Enable/test backups separately
```

For the first deployment, running the stages individually is preferable to immediately running `site.yml`.

Example:

```bash
ansible-playbook playbooks/prepare.yml
```

Then:

```bash
ansible-playbook playbooks/deploy.yml
```

Then:

```bash
ansible-playbook playbooks/harden.yml
```

Finally, when backups have been configured:

```bash
ansible-playbook playbooks/backup.yml
```

---

# 27. Post-Deployment Verification

After deployment, verify Docker:

```bash
docker info
```

Verify the GitLab container:

```bash
docker ps --filter name=gitlab
```

Check its health:

```bash
docker inspect --format '{{.State.Health.Status}}' gitlab
```

Expected:

```text
healthy
```

Check the container configuration:

```bash
docker inspect gitlab
```

Important properties to verify include:

```text
Privileged = false
NetworkMode = bridge
```

Verify published ports:

```bash
docker port gitlab
```

Verify GitLab health:

```bash
curl -k https://gitlab.example.internal/-/health
```

Replace the example hostname with the actual GitLab FQDN.

---

# 28. Firewall Verification

Check UFW:

```bash
sudo ufw status verbose
```

Verify the expected ports:

```text
22/tcp
443/tcp
2222/tcp
```

Check the Docker firewall chain:

```bash
sudo iptables -nL GITLAB-DOCKER-USER
```

Check the Docker `DOCKER-USER` chain:

```bash
sudo iptables -nL DOCKER-USER
```

---

# 29. IPv6 Verification

Check the kernel settings:

```bash
sysctl net.ipv6.conf.all.disable_ipv6
sysctl net.ipv6.conf.default.disable_ipv6
sysctl net.ipv6.conf.lo.disable_ipv6
```

Expected:

```text
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```

Verify UFW:

```bash
grep '^IPV6=' /etc/default/ufw
```

Expected:

```text
IPV6=no
```

Verify IPv6 addresses:

```bash
ip -6 addr
```

The host should not have normal operational IPv6 addresses when IPv6 is disabled.

---

# 30. TLS Verification

Inspect the generated certificate:

```bash
openssl x509 \
  -in /home/<deployment-user>/gitlab/config/tls/gitlab.crt \
  -noout \
  -subject \
  -issuer \
  -dates \
  -ext subjectAltName \
  -text
```

Verify the certificate uses the expected FQDN and contains the expected SAN.

Verify the RSA key:

```bash
openssl rsa \
  -in /home/<deployment-user>/gitlab/config/tls/gitlab.key \
  -check \
  -noout
```

The private key should not be readable by ordinary users.

---

# 31. GitLab Configuration Verification

GitLab application settings can be inspected from the container.

Example:

```bash
docker exec --user root gitlab gitlab-rails runner \
  'puts Gitlab::CurrentSettings.current_application_settings.to_json'
```

The hardening role also performs automated verification of the configured security policies.

---

# 32. Updating GitLab

GitLab upgrades should be treated as controlled configuration changes.

Do not simply change:

```yaml
gitlab_image_version:
```

without updating the corresponding image digest.

A controlled upgrade should include:

```text
1. Select GitLab version
2. Verify release compatibility
3. Obtain the corresponding image digest
4. Update version
5. Update digest
6. Review GitLab configuration changes
7. Validate Ansible
8. Back up GitLab
9. Deploy
10. Verify GitLab health
11. Verify application functionality
```

Example:

```yaml
gitlab_image_version: "NEW_VERSION"
gitlab_image_digest: "sha256:NEW_DIGEST"
```

The actual digest must correspond to the intended GitLab image.

---

# 33. Rollback Considerations

Rollback should not be treated as simply changing the Docker image tag.

GitLab upgrades can include database migrations and application data changes.

A rollback strategy should therefore consider:

* GitLab version compatibility
* database migrations
* application data
* configuration
* uploaded repositories and artifacts
* backups
* restore procedures

A tested backup/restore process is required before relying on rollback as a disaster-recovery mechanism.

---

# 34. Secrets and Sensitive Configuration

Sensitive values should not be committed directly into:

```text
group_vars/gitlab.yml
```

The repository contains:

```text
group_vars/gitlab_vault.yml
```

for sensitive Ansible variables.

Use Ansible Vault for secrets:

```bash
ansible-vault encrypt group_vars/gitlab_vault.yml
```

To edit:

```bash
ansible-vault edit group_vars/gitlab_vault.yml
```

To run a playbook using the vault:

```bash
ansible-playbook playbooks/site.yml --ask-vault-pass
```

Do not commit plaintext passwords, private keys, tokens, or other secrets.

---

# 35. File Permissions

The project intentionally uses restrictive permissions for sensitive GitLab configuration and TLS material.

Examples include:

```text
TLS directory:
root:root
0700

TLS private key:
root:root
0600

GitLab configuration:
root:root
0600
```

The Docker Compose file is owned by the deployment user with restricted permissions.

---

# 36. Operational Principles

The project follows several operational principles.

### Do not manually modify generated configuration

GitLab configuration is generated by Ansible.

Manual changes to generated files can be overwritten during subsequent deployments.

Configuration changes should normally be made in:

```text
group_vars/gitlab.yml
```

or the relevant role/template.

### Do not expose unnecessary ports

New published ports should not be added without also reviewing:

* Docker Compose
* UFW
* Docker `DOCKER-USER`
* GitLab application configuration

### Do not mount the Docker socket

The GitLab container should not receive:

```text
/var/run/docker.sock
```

### Do not use privileged containers

The GitLab container is intentionally configured with:

```yaml
privileged: false
```

### Keep the GitLab image pinned

Both version and digest should be controlled.

---

# 37. Troubleshooting

## Ansible cannot connect

Test SSH directly:

```bash
ssh -p 2222 <ansible-user>@<server-ip>
```

Then:

```bash
ansible gitlab -m ansible.builtin.ping
```

Check the inventory:

```bash
ansible-inventory --graph
```

---

## Docker is not running

Check:

```bash
sudo systemctl status docker
```

Check:

```bash
sudo journalctl -u docker --no-pager -n 100
```

Validate the daemon configuration:

```bash
sudo cat /etc/docker/daemon.json
```

---

## GitLab container is unhealthy

Check:

```bash
docker ps -a --filter name=gitlab
```

Then:

```bash
docker logs --tail 200 gitlab
```

Check the health state:

```bash
docker inspect \
  --format '{{json .State.Health}}' \
  gitlab
```

---

## GitLab reconfiguration failed

Check:

```bash
docker exec --user root gitlab gitlab-ctl status
```

Then:

```bash
docker exec --user root gitlab gitlab-ctl reconfigure
```

Review the resulting error before changing the Ansible configuration.

---

## UFW blocks required access

Check:

```bash
sudo ufw status verbose
```

Check the Docker firewall:

```bash
sudo iptables -nL DOCKER-USER
```

Check:

```bash
sudo iptables -nL GITLAB-DOCKER-USER
```

Remember that Docker-published traffic is subject to Docker's own packet-filtering path and therefore should be investigated separately from ordinary UFW INPUT rules.

---

# 38. Security Validation Checklist

Before considering the deployment production-ready, verify:

```text
[ ] Ubuntu version is correct
[ ] CPU architecture is x86_64
[ ] CPU and memory requirements are satisfied
[ ] Administrative SSH uses port 2222
[ ] GitLab SSH uses port 22
[ ] GitLab HTTPS uses port 443
[ ] UFW is active
[ ] UFW inbound policy is deny
[ ] Required inbound ports are explicitly allowed
[ ] Docker firewall chain is active
[ ] IPv6 is disabled
[ ] GitLab container is not privileged
[ ] Docker socket is not mounted
[ ] Docker bridge networking is used
[ ] no-new-privileges is enabled
[ ] GitLab image uses the expected version
[ ] GitLab image digest is verified
[ ] TLS certificate uses the correct FQDN
[ ] TLS private key has restrictive permissions
[ ] TLS 1.2/1.3 are configured
[ ] HSTS is enabled
[ ] GitLab signup is disabled
[ ] Mandatory 2FA is enabled
[ ] 2FA grace period matches policy
[ ] Unknown sign-in notifications are enabled
[ ] PAT expiration policy is enabled
[ ] GitLab health endpoint reports healthy
[ ] GitLab application settings have been verified
[ ] Backup strategy has been defined
[ ] Restore procedure has been tested
```

---

# 39. Current Defaults

The current deployment intentionally starts with:

```text
GitLab:
  Edition: EE
  Version: 19.3.1-ee.0

TLS:
  Enabled: Yes
  Type: Self-signed
  RSA: 4096-bit
  Validity: 825 days

Network:
  GitLab HTTPS: 443
  GitLab SSH: 22
  Administration SSH: 2222
  IPv6: Disabled

Firewall:
  UFW: Enabled
  Default inbound: Deny
  Docker firewall: Enabled

GitLab registration:
  Disabled

GitLab admin mode:
  Enabled

Mandatory 2FA:
  Enabled

Unknown sign-in notifications:
  Enabled

PAT expiry:
  Required

GitLab Registry:
  Disabled

GitLab Pages:
  Disabled

Backups:
  Disabled initially

Docker:
  Privileged: No
  Docker socket: Not mounted
  Network: Bridge
  no-new-privileges: Enabled
```

---

# 40. Design Decisions

The following decisions are intentional.

### Ubuntu Docker packages

Docker is installed from Ubuntu repositories rather than maintaining a separate upstream Docker APT repository.

This reduces external repository and signing-key management.

### Docker bridge networking

The GitLab container uses normal Docker bridge networking rather than host networking.

This keeps container network exposure explicit through published ports.

### No Docker socket

The GitLab container does not require access to the Docker daemon.

Therefore the Docker socket is not mounted.

### IPv6 disabled

IPv6 is disabled because the current deployment does not require it and the project is designed around an IPv4-only network policy.

### Self-signed TLS

Self-signed TLS is used for the initial deployment so that the project does not depend on an external certificate authority or Let's Encrypt.

Production environments can replace this with an organizational PKI.

### Backups disabled initially

Backups are implemented separately and remain disabled until the operational backup and restore strategy has been deliberately configured and tested.

---

# 41. Known Scope and Limitations

This project currently focuses on deploying and hardening a single GitLab instance.

It does not currently provide:

* GitLab Geo
* High availability
* Multi-node GitLab
* External PostgreSQL
* External Redis
* External object storage
* Automated disaster recovery
* Automated certificate renewal
* Production PKI integration
* Kubernetes-based GitLab deployment
* Automated GitLab version upgrade selection
* Automated restore testing
* Monitoring infrastructure
* Centralized SIEM integration

These should be treated as separate architectural extensions rather than silently introduced into the current deployment.

---

# 42. Change Management

Before changing the deployment:

```bash
git status
git diff
```

Review the intended changes.

Then validate:

```bash
git diff --check
ansible-playbook --syntax-check playbooks/site.yml
ansible-lint
```

For infrastructure changes, test the affected playbook or role before running the complete deployment.

Commit meaningful changes with descriptive messages.

Example:

```bash
git add .
git commit -m "Disable IPv6 and align UFW policy"
```

Then push:

```bash
git push origin main
```

---

# 43. Recommended First Deployment

The first deployment should be performed incrementally.

Start with:

```bash
ansible-playbook playbooks/prepare.yml
```

Verify the host.

Then:

```bash
ansible-playbook playbooks/deploy.yml
```

Verify GitLab.

Then:

```bash
ansible-playbook playbooks/harden.yml
```

Verify GitLab security settings.

Keep backups disabled until the deployment has been validated.

After the backup and restore process has been tested, enable:

```yaml
gitlab_backup_enabled: true
```

and run:

```bash
ansible-playbook playbooks/backup.yml
```

---

# 44. Project Status

The project currently provides the core deployment architecture for a single hardened GitLab EE instance.

Before production deployment, the repository should pass:

```text
Ansible syntax validation
Ansible Lint validation
Compose configuration validation
SSH connectivity validation
Host preparation validation
Docker validation
Firewall validation
GitLab health validation
GitLab security-policy validation
TLS validation
Backup/restore validation
```

The deployment should not be considered production-ready solely because Ansible completes successfully. Runtime and security verification are required after deployment.

---

# 45. License

Add the project's applicable license here.

Example:

```text
This project is licensed under the MIT License.
```

# Configuration Ownership and Precedence

Configuration is intentionally divided between inventory, group variables, role defaults, role tasks, templates, and handlers.

The general rule is:

> Define deployment policy in `group_vars/gitlab.yml`; use role defaults only for reusable defaults; implement behavior in roles; do not duplicate the same policy in multiple locations.

## Configuration ownership

| Configuration           | Owner                           | Purpose                                        |
| ----------------------- | ------------------------------- | ---------------------------------------------- |
| Target hostname/IP      | `group_vars/gitlab.yml`         | Ansible connection target                      |
| Ansible SSH user        | `group_vars/gitlab.yml`         | Administrative account                         |
| Ansible SSH port        | `group_vars/gitlab.yml`         | Host administration SSH port                   |
| GitLab FQDN             | `group_vars/gitlab.yml`         | Deployment-specific hostname                   |
| GitLab version          | `group_vars/gitlab.yml`         | Selected GitLab release                        |
| GitLab image digest     | `group_vars/gitlab.yml`         | Image integrity/pinning                        |
| GitLab HTTPS port       | `group_vars/gitlab.yml`         | External GitLab HTTPS port                     |
| GitLab SSH port         | `group_vars/gitlab.yml`         | External GitLab SSH port                       |
| TLS policy              | `group_vars/gitlab.yml`         | Deployment-specific certificate policy         |
| GitLab security policy  | `group_vars/gitlab.yml`         | 2FA, signup, PAT, admin mode, etc.             |
| IPv6 policy             | `group_vars/gitlab.yml`         | Whether IPv6 is disabled                       |
| Firewall allowed ports  | `group_vars/gitlab.yml`         | Deployment-specific network exposure           |
| Backup enablement       | `group_vars/gitlab.yml`         | Whether backup automation is enabled           |
| Sensitive values        | `group_vars/gitlab_vault.yml`   | Secrets managed with Ansible Vault             |
| Generic role defaults   | `roles/*/defaults/main.yml`     | Safe reusable role defaults                    |
| Host configuration      | `roles/host_prepare/`           | Packages, sysctl, filesystem preparation       |
| Docker configuration    | `roles/docker/`                 | Docker installation and daemon configuration   |
| Firewall implementation | `roles/firewall/`               | UFW and Docker firewall behavior               |
| GitLab implementation   | `roles/gitlab/`                 | TLS, Compose, container, GitLab configuration  |
| GitLab hardening        | `roles/gitlab/tasks/harden.yml` | Application-level security enforcement         |
| Backup implementation   | `roles/gitlab_backup/`          | Backup service and timer                       |
| Generated configuration | Role templates                  | Runtime configuration generated from variables |

## Source of deployment policy

`group_vars/gitlab.yml` is the primary source of truth for this single-host GitLab deployment.

For example:

```yaml
gitlab_fqdn: "gitlab.example.internal"
gitlab_https_port: 443
gitlab_ssh_port: 22
gitlab_disable_ipv6: true
gitlab_signup_enabled: false
gitlab_admin_mode_enabled: true
gitlab_2fa_required: true
gitlab_backup_enabled: false
```

These values describe what the deployment should look like.

The roles are responsible for implementing that policy.

For example:

```text
gitlab_disable_ipv6
        |
        v
host_prepare
        |
        +--> kernel sysctl configuration

firewall_manage_ipv6
        |
        v
firewall
        |
        +--> UFW IPV6 setting
```

The role should not independently redefine the deployment policy.

## Role defaults

Files under:

```text
roles/*/defaults/main.yml
```

provide reusable defaults for a role.

They should be used for values that are reasonable across deployments and that do not represent site-specific policy.

For example:

```yaml
docker_service_enabled: true
docker_validate_installation: true
gitlab_restart_policy: unless-stopped
gitlab_container_name: gitlab
```

Deployment-specific values belong in `group_vars/gitlab.yml`.

Avoid defining the same variable with different intended values in both locations.

## Ansible variable precedence

Ansible applies variable precedence when the same variable is defined in multiple places.

For this project, the practical rule is:

```text
higher precedence
        |
        v
extra vars (-e)
        |
inventory/group/host variables
        |
group_vars
        |
role defaults
        |
lower precedence
```

The exact Ansible precedence model is more detailed than this simplified representation, but the important operational principle is:

> Do not rely on precedence to resolve conflicting configuration accidentally.

If a variable is intended to have one deployment-wide value, define it once in the appropriate ownership layer.

## Extra variables

Extra variables supplied with:

```bash
ansible-playbook playbooks/site.yml -e "variable=value"
```

have very high precedence.

They can therefore override normal configuration.

This should be used deliberately, primarily for controlled testing or temporary overrides.

For permanent deployment policy, modify the appropriate configuration file instead of relying on `-e`.

For example, do not permanently configure:

```bash
ansible-playbook playbooks/site.yml -e "gitlab_backup_enabled=true"
```

Instead, change:

```yaml
gitlab_backup_enabled: true
```

in the appropriate variables file.

## Secrets

Secrets have a separate ownership boundary.

Non-sensitive deployment policy belongs in:

```text
group_vars/gitlab.yml
```

Sensitive values belong in:

```text
group_vars/gitlab_vault.yml
```

The latter should be encrypted with Ansible Vault.

Example:

```bash
ansible-vault edit group_vars/gitlab_vault.yml
```

Do not move ordinary configuration into Vault merely because Vault is available. Keeping non-sensitive configuration visible makes the deployment easier to audit.

## Inventory versus group variables

The inventory identifies hosts.

For example:

```yaml
all:
  children:
    gitlab:
      hosts:
        gitlab-server:
```

Deployment-specific connection information is currently maintained in:

```text
group_vars/gitlab.yml
```

This keeps the inventory intentionally minimal.

If the project is later expanded to multiple GitLab hosts, host-specific connection variables can be moved into the inventory or host-specific variable files as appropriate.

## Templates do not own policy

Jinja templates under:

```text
roles/*/templates/
```

generate configuration files.

They should consume variables rather than establish independent policy.

For example, the Docker Compose template should use:

```jinja2
{{ gitlab_https_port }}
```

rather than independently defining:

```text
443
```

This keeps the rendered configuration synchronized with the firewall and GitLab configuration.

## Tasks implement policy

Ansible tasks are responsible for enforcing the configuration represented by variables.

For example:

```text
group_vars/gitlab.yml
        |
        v
gitlab_disable_ipv6: true
        |
        v
host_prepare task
        |
        v
sysctl configuration
```

A task should not silently override a centrally defined policy.

## Handlers implement lifecycle actions

Handlers are used for actions that should occur when configuration changes.

Examples include:

```text
Docker daemon restart
GitLab reconfiguration
GitLab restart
Docker firewall service restart
```

Handlers should not normally contain the source of configuration policy. They react to changes produced by tasks.

The GitLab role explicitly flushes pending handlers before performing its final health validation:

```text
configuration
     |
     +--> notify handler
     |
container deployment
     |
     +--> flush_handlers
     |
reconfigure/restart
     |
health validation
```

This ensures that the health check is performed after required lifecycle actions.

## Avoiding duplicate configuration

When reviewing a change, first determine which layer owns the setting.

For example, if a firewall port is already defined in:

```text
group_vars/gitlab.yml
```

do not independently add the same port to:

```text
roles/firewall/defaults/main.yml
```

or hard-code it in a firewall template.

Likewise, if IPv6 policy is defined by:

```yaml
gitlab_disable_ipv6: true
```

the implementation should consume that policy rather than introducing a second independent GitLab-specific IPv6 switch.

## Recommended change procedure

When changing configuration:

1. Identify the owner of the setting.
2. Change the policy at the owning layer.
3. Verify the relevant role consumes that variable.
4. Verify templates use the variable rather than a hard-coded value.
5. Verify dependent configuration remains consistent.
6. Run syntax and lint validation.
7. Review the rendered/runtime result.

The objective is to maintain a single authoritative policy value and have the implementation layers consume it consistently.
