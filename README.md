# Docker Swarm cluster provisioning

> Repository contains everything to provision a Docker Swarm cluster.

<br>

**TOC**:

- [1. Requirements](#1-requirements)
- [2. Proxy requirements](#2-proxy-requirements)
- [3. Configuration](#3-configuration)
- [4. Installation](#4-installation)
  - [4.1 Docker](#4-1-docker)
  - [4.2 Swarm Cluster](#4-2-swarm-cluster)
- [5. Verification](#5-verification)
- [6. Management tools](#6-management-tools)
  - [6.1 Portainer and Traefik](#6-1-portainer-and-traefik)
    - [6.1.1 Post installation configuration](#6-1-1-post-installation-configuration)
  - [6.2 Gitea](#6-2-gitea)
    - [6.2.1 Post installation configuration](#6-2-1-post-installation-configuration)
    - [6.2.2 Notification via MS Teams](#6-2-2-notification-via-ms-teams)
- [7. Upgrade](#7-upgrade)
  - [7.1 Docker](#7-1-docker)
  - [7.2 Management tools](#7-2-management-tools)
- [8. Uninstall](#8-uninstall)

<br>

## 1. Requirements

**System requirements**:
- 1 - 4 VMs with minimum of 2 cpus and 4GB RAM each (HDD size is on your preference)
- static IP on all VMs
- supported linux OS: Ubuntu 20/22.04, RedHat/CentOS 7/8, Rocky 8

**Workstation requirements**:
- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)

**Additional requirements**:
- NFS share for shared docker volumes on all hosts
- [VMs OS and Workstation requirements](docs/requirements.md)
- multi manager cluster requires additional load balancer in front of the cluster (HaProxy, NetScaler, etc)

<br>

## 2. Proxy requirements

Access to following external resources are required:

- Access to OS repository (yum/apt)
- Access to Python packages (*.python.org, *.pypi.org, *.pythonhosted.org)
- Access to Docker packages and images (*.docker.com, *.docker.io)

<br>

## 3. Configuration

- Create copy of `inventory/default` directory and name it after your environment, example: `inventory/<environment>`
- Edit your environment `inventory.ini` file by filling out environment specific configuration values.
- Edit your environment `all-vault.yml` file by filling out environment specific secrets.

> NOTE: Encrypt the `all-vault.yaml` file using `ansible-vault encrypt inventory/<environment>/group_vars/all/all-vault.yml` and providing secure password.

**Inventory file sample**:

```ini
# Server hostnames <environment>-<role>-<index>
[all]
ds-manager-01 ansible_host=127.0.0.11
ds-manager-02 ansible_host=127.0.0.12
ds-worker-01 ansible_host=127.0.0.13
ds-worker-02 ansible_host=127.0.0.14

# Common variables for all hosts
[all:vars]
# User for ssh connections
ansible_user=
# Users private key for passwordless connection
ansible_ssh_private_key_file=~/.ssh/id_rsa
# Become method
ansible_become_method=sudo
# SSH common arguments
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
# Proxy configuration
; http_proxy=
; https_proxy=
; no_proxy=10.0.0.0/8,172.16.0.0/12,192.168.0.0/16

# Base URL for all kind of deployments, example: gitea.example.com
# NOTE: URL will be used for management deployments and for Gitea
base_hostname=gitea.example.com

# Shared volume settings for gitea deployment
share_enable=true
share_type=nfs
share_server=127.0.0.10
share_server_path=/docker_volume

[manager]
ds-manager-01
ds-manager-02

[worker]
ds-worker-01
ds-worker-02

[swarm:children]
manager
worker
```

**Vault file sample**:

```yaml
#############################################################################################
# Encrypt the file using strong password:
#   ansible-vault encrypt inventory/group_vars/all/all-vault.yml
#
# Passwords with special characters might not work correctly.
# Valid passwords can be created using the following command:
#   tr -cd '[:alnum:]' < /dev/urandom | fold -w16 | head -n1
#
# Files can be encoded in base64 format using the following command:
#   base64 -w 0 <file>
#############################################################################################

# Certificate and private key in base64 format
# NOTE: TLS certificates should be issued by trusted certificate authority.
traefik_tls_crt:
traefik_tls_key:

# Traefik credentials
vault_traefik_user:
vault_traefik_pass:
```

<br>

## 4. Installation

> :warning: Cluster installation is divided into several steps to make full use of playbooks for different needs.

To test SSH connection to each VM is working, simply run this playbook to verify.

```shell
ansible-playbook -i inventory/<environment>/inventory.ini playbooks/check.yml
```

### 4.1 Docker

Playbook to install and configure Docker.

```shell
ansible-playbook -i inventory/<environment>/inventory.ini playbooks/docker.yml
```

### 4.2 Swarm Cluster

Playbook will convert Standalone Docker installation into a Swarm mode cluster.

```shell
ansible-playbook -i inventory/<environment>/inventory.ini playbooks/init-cluster.yml
```

Will configure the cluster based on provided configuration in inventory files.

<br>

## 5. Verification

SSH into one of the cluster manager nodes and execute following command:

```shell
docker node ls
```

Output should look like this:

```
ID                            HOSTNAME        STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
sltz1m28nda5na01ebv00y5db *   ds-manager-01   Ready     Active         Leader           26.0.0
aceu43odqfz1eh98ffjb6ucrs     ds-manager-02   Ready     Active         Reachable        26.0.0
z0gwvo1s7l4u27nfrm8hhjeay     ds-worker-01    Ready     Active                          26.0.0
```

<br>

## 6. Management tools

> This section includes most popular management tools for a Docker Swarm Cluster

### 6.1 Portainer and Traefik

Playbook to setup cluster management tool deployments:
1. Portainer
2. Traefik

```shell
ansible-playbook -i inventory/<environment>/inventory.ini playbooks/management.yml --ask-vault-pass
```

<br>

#### 6.1.1 Post installation configuration

In following sections replace `{{ variable }}` with the configured value of `variable`.

**Portainer**:

1. In browser navigate to `https://{{ base_hostname }}/portainer/`
2. Go through: "New Portainer installation"
   - Please create the initial administrator user
      - Provide your password (`The password must be at least 12 characters long`)
      - Confirm password
      - Uncheck "Allow collection of anonymous statistics. You can find more information about this in our privacy policy."
      - Click "Create user"

**Traefik**:

1. In browser navigate to `https://{{ base_hostname }}/traefik/dashboard/`.
2. Login with credentials provided in `all-vault.yml` file:
    ```
    vault_traefik_user:
    vault_traefik_pass:
    ```
3. Navigate to "HTTP" and click on "HTTP Routers" tab.
4. Verify that Name: `traefik@docker` and `portainer@docker` are listed in table.

<br>

### 6.2 Gitea

Playbook to setup Gitea deployment.

Change values in [roles/stacks/gitea/defaults/main.yml](roles/stacks/gitea/defaults/main.yml) file.

After successful "New Portainer installation" run Gitea deployment setup playbook.

You have to pass `--extra-vars=` key with `"key1=value1 key2=value2"` variables to the playbook:
- `portainer_user=` - Portainer admin username.
- `portainer_password=` - Portainer admin user password.
- `db_type=` - Gitea database type, example: `postgres`, `mysql`, `mssql` or `sqlite3`. If you set type to `sqlite3`, then rest database configuration can be ignored.
- `db_host=` - Gitea database host.
- `db_port=` - Gitea database port.
- `db_name=` - Gitea database name.
- `db_user=` - Gitea database username.
- `db_password=` - Gitea database password.

Run playbook:

```shell
ansible-playbook -i inventory/<environment>/inventory.ini playbooks/gitea.yml --extra-vars "portainer_user='admin' portainer_password='password' db_type='mysql' db_host='localhost' db_port='3306' db_name='gitea' db_user='gitea' db_password='password'"
```

If you set `db_type` to `sqlite3`:

```shell
ansible-playbook -i inventory/<environment>/inventory.ini playbooks/gitea.yml --extra-vars "portainer_user='admin' portainer_password='password' db_type='sqlite3'"
```

#### 6.2.1 Post installation configuration

In following sections replace `{{ variable }}` with the configured value of `variable`.

1. In browser navigate to `https://{{ base_hostname }}/`
2. In "Initial Configuration" section:
   - check that all fields are filled with correct values.
3. In "Optional Settings" section:
   - expand "Email Settings":
     - check that all fields are filled with correct values.
     - check "Enable Email Notifications" checkbox if needed.
    - expand "Server and Third-Party Service Settings":
      - check "Disable gravatar" checkbox.
    - expand "Administrator Account Settings":
      - fill out all fields required for administrator account.
4. Click "Install Gitea" button.
5. Wait for 10 seconds and refresh the page.

#### 6.2.2 Notification via MS Teams

This is optional step.

Follow the guide written in [docs/notifications.md](docs/notifications.md) file to setup notification via MS Teams.

<br>

## 7. Upgrade

> This section contains instructions to upgrade cluster components and deployments

### 7.1 Docker

For upgrading docker please change `docker_version` variable's value to newer docker version in [all.yml](inventory/default/group_vars/all/all.yml)

Example:

```yaml
docker_version: "24.0.0"

# to

docker_version: "26.0.0"
```

And run upgrade playbook:

```shell
ansible-playbook -i inventory/<environment>/inventory.ini playbooks/docker-upgrade.yml
```

### 7.2 Management tools

To upgrade cluster management and gitea deployments you have to re-run setup playbooks.

Provide new deployment versions in file `inventory/<environment>/group_vars/all/all.yml` for variables:

```yml
portainer_version:
traefik_version:
gitea_version:
```

For management deployments upgrade, simply re-run setup playbook with providing role tag:

```shell
ansible-playbook -i inventory/<environment>/inventory.ini playbooks/management.yml --ask-vault-pass --tags "mgmt"
```

For gitea deployment upgrade, simply re-run gitea playbook as its described in [6.2 Gitea](#62-gitea) section.

<br>

## 8. Uninstall

Playbook to uninstall Docker Swarm cluster.

```shell
ansible-playbook -i inventory/<environment>/inventory.ini reset.yml
```

Will uninstall the Docker Swarm cluster and reboot the machines.
