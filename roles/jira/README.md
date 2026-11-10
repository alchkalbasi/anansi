<p><img src="./jira-logo.svg" alt="jira logo" title="Jira" align="right" height="60" /></p>

# Ansible Role: Jira Service

## Description

This role automates the deployment of **Jira Software** using Docker and Docker Compose.
It prepares the service directory, templates the environment file, deploys Jira with a
dedicated PostgreSQL database and a backup sidecar, and waits for Jira to become healthy.

After deployment, finish the setup in the Jira web UI and enter your own Atlassian license
(a free trial or Data Center license from <https://my.atlassian.com>).

Optionally, it installs a cron job that syncs the local backups to a remote server over SSH.

---

## Features

- Verifies Docker CE and the Docker Compose plugin are installed.
- Creates the public and private Docker networks shared with Traefik.
- Deploys Jira, PostgreSQL, and a backup container with Docker Compose.
- Takes periodic `pg_dump` and Jira home backups, pruning old ones.
- Waits for the Jira container health check to pass.
- Optionally syncs backups to a remote host with `rsync` from cron.

---

## Requirements

- Docker CE and the Docker Compose plugin on the target host.
- The `community.docker` collection.
- A valid Atlassian license for Jira Software, entered in the setup wizard.
- Traefik running on `jira_docker_public_network` with the `jira_traefik_entrypoint`
  entrypoint and the `jira_traefik_certresolver` resolver.

---

## Tags

| Tag              | Runs                                    |
| ---------------- | --------------------------------------- |
| `setup_jira`     | Docker checks, files, pull, and deploy  |
| `wait_jira`      | Wait for the Jira health check          |
| `backup`, `cron` | Remote backup sync (when enabled)       |

---

## Variables

### Main variables

```yaml
jira_domain: example.com
jira_subdomain: jira
jira_fqdn: "{{ jira_subdomain }}.{{ jira_domain }}"
jira_project_dir: /opt/services
jira_service_dir: "{{ jira_project_dir }}/jira"
jira_image_tag: "10.3.21-jdk17"
jira_postgres_image_tag: "16-alpine"
jira_container_name: "{{ jira_hostname }}"
jira_http_bind_address: 127.0.0.1
jira_http_bind_port: 8080
jira_jvm_minimum_memory: "2048m"
jira_jvm_maximum_memory: "3072m"

jira_docker_private_network: prinet
jira_docker_public_network: pubnet
jira_traefik_entrypoint: web-secure
jira_traefik_certresolver: le
```

### Local backups

```yaml
jira_backup_init_sleep: 90s
jira_backup_interval: 86400
jira_postgres_backup_prune_days: 7
jira_data_backup_prune_days: 7
```

### Remote backup sync

```yaml
jira_remote_backup_enabled: false
jira_remote_backup_user: backup
jira_remote_backup_host: ""
jira_remote_backup_port: 22
jira_remote_backup_path: /backups/jira
jira_remote_backup_identity_file: /root/.ssh/id_ed25519
jira_remote_backup_known_hosts: []   # - { name: backup.example.com, key: "backup.example.com ssh-ed25519 AAAA..." }
jira_remote_backup_cron_minute: "17"
jira_remote_backup_cron_hour: "04"
```

### Secret variables

Override these from vaulted inventory vars, not from role defaults:

```yaml
jira_db_name: x
jira_db_user: x
jira_db_password: x
```
