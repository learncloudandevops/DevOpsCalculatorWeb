
# Calculator Ansible Deployment

This directory contains Ansible playbooks for preparing the Calculator
application server and deploying the Calculator Web application.

## Files

| File | Purpose |
|---|---|
| `ansible.cfg` | Ansible defaults and SSH host-key verification |
| `inventory.ini` | Calculator target EC2 instance |
| `install-java.yml` | Installs Java 21 on Amazon Linux 2023 |
| `install-tomcat.yml` | Installs and configures Tomcat |
| `deploy-calculator.yml` | Downloads the WAR from Nexus and deploys it |
| `nexus-vault.yml` | Encrypted Nexus credentials; maintained separately |

## Prerequisites

- Ansible installed on the Jenkins EC2 instance.
- Network and SSH connectivity from Jenkins to the Tomcat EC2 instance.
- A valid SSH private-key credential in Jenkins.
- A trusted SSH host key for the target server.
- Nexus credentials with permission to download the published artifact.
- An encrypted `nexus-vault.yml` file.
- A Jenkins secret-text credential named `ansible-vault-password`.

## Configure the inventory

Update `inventory.ini` with the private IP address of the Tomcat EC2 instance.

## Install Java and Tomcat

Run these playbooks from the repository root after configuring SSH access:

```bash
ansible-playbook -i ansible/inventory.ini ansible/install-java.yml

ansible-playbook -i ansible/inventory.ini ansible/install-tomcat.yml
```

The Tomcat installation playbook must match the service name, installation
directory, and service account used in your environment.

## Deploy the application

The deployment playbook expects `artifact_version` and reads Nexus credentials
from the encrypted Vault file.

```bash
ansible-playbook \
  -i ansible/inventory.ini \
  ansible/deploy-calculator.yml \
  --ask-vault-pass \
  -e "artifact_version=YOUR_MAVEN_VERSION"
```

For Jenkins, provide the Vault password through the configured Jenkins
credential rather than storing it in the repository.

## Security

- Commit only the encrypted `nexus-vault.yml` file.
- Never commit plaintext Nexus credentials or the Vault password.
- Keep SSH host-key checking enabled.
- Do not place private SSH keys in this repository.
