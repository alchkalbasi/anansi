# Garage Deployment Stack

An **Ansible role** paired with a **Docker Compose** stack that deploys a self-hosted, single-node [Garage](https://garagehq.deuxfleurs.fr/) S3-compatible object store.

The role checks the target host, prepares directories, generates and keeps secrets, templates `garage.toml`, starts the container, and then sets up the cluster layout, buckets and access keys.

---

## Prerequisites

* **Operating system:** Linux (Ubuntu/Debian recommended)
* **On the host:** `docker-ce` and `docker-compose-plugin` (see the `docker` role)
* **On the control node:** the `community.docker` collection

---

## Layout on the host

```
/opt/services/garage/
├── compose.yml
├── .env
├── garage.toml
├── meta/                 # metadata DB (LMDB) + snapshots
├── data/                 # object blocks
└── secrets/              # 0700, mounted read-only into the container
    ├── rpc_secret
    ├── admin_token
    ├── metrics_token
    └── keys/<name>.txt   # credentials of keys created by this role
```

---

## Variables

| Variable | Default | Description |
| --- | --- | --- |
| `service_dir` | `/opt/services/garage` | Where the stack is deployed |
| `docker_networks` | `[prinet, pubnet]` | External networks to create. Garage joins the first one |
| `garage_image_tag` | `v2.5.0` | `dxflrs/garage` image tag |
| `garage_bind_host` | `127.0.0.1` | Host IP the ports bind to. Use `0.0.0.0` to expose directly |
| `garage_s3_api_port` / `garage_s3_web_port` / `garage_admin_port` | `3900` / `3902` / `3903` | Published host ports |
| `garage_db_engine` | `lmdb` | `lmdb` or `sqlite` |
| `garage_replication_factor` | `1` | Keep `1` for a single node |
| `garage_s3_region` | `garage` | S3 region name clients must use |
| `garage_s3_root_domain` / `garage_web_root_domain` | `.s3.garage.localhost` / `.web.garage.localhost` | Domains for vhost-style buckets and static websites |
| `garage_rpc_secret` / `garage_admin_token` / `garage_metrics_token` | `""` | Empty means a random value is generated once on the host. Set them (with ansible-vault) to pin values |
| `garage_webui_enabled` | `true` | Run [garage-webui](https://github.com/khairul169/garage-webui) next to Garage (compose profile `webui`) |
| `garage_webui_image_tag` | `1.1.0` | `khairul169/garage-webui` image tag |
| `garage_webui_bind_host` / `garage_webui_port` | `{{ garage_bind_host }}` / `3909` | Where the Web UI is published |
| `garage_webui_auth_user_pass` | `""` | Basic auth as `user:<bcrypt>` (`htpasswd -nbBC 10 admin '<password>'`). Required unless bound to `127.0.0.1` |
| `garage_layout_zone` / `garage_layout_capacity` | `dc1` / `100G` | Layout role given to the node on first run |
| `garage_buckets` | `[]` | Bucket names to create (S3 naming rules: 3–63 chars, lowercase) |
| `garage_keys` | `[]` | Access keys and their bucket permissions (see below) |

Example:

```yaml
garage_layout_capacity: 500G
garage_buckets:
  - backups
  - media
garage_keys:
  - name: restic
    buckets:
      - name: backups
        permissions: [read, write]
  - name: webapp
    buckets:
      - name: media
        permissions: [read, write, owner]
```

---

## Usage

```bash
ansible-playbook -i inventory.ini playbook.yml --tags garage
```

Tags you can use to run only some steps:

| Tag | What it does |
| --- | --- |
| `check-docker` | Checks that Docker and Compose are installed |
| `prepare` | Creates directories and copies `compose.yml` |
| `configure` | Writes secrets, `.env` and `garage.toml` |
| `deploy` | Creates the networks, pulls the image and runs `docker compose up` |
| `initialize` | Sets up the layout, buckets and keys |
| `buckets` / `keys` | Runs only the bucket or key steps |

---

## After deployment

```bash
# Cluster state
docker exec garage /garage status

# Credentials of a key created by the role
sudo cat /opt/services/garage/secrets/keys/restic.txt

# Test with the AWS CLI
aws --endpoint-url http://127.0.0.1:3900 --region garage s3 ls
```

> 🔐 A key's secret is only shown once, when the key is created. The role saves it to `secrets/keys/<name>.txt`. If you lose that file, run `garage key info --show-secret <name>` or create a new key.

> 🌐 The ports are bound to `127.0.0.1` by default. To serve S3 publicly, put Traefik or nginx in front of port `3900`, or set `garage_bind_host: 0.0.0.0` and open the port in your firewall.

> ➕ **Scaling out:** set `garage_rpc_public_addr` to `<host_ip>:3901`, publish port 3901, share the same `garage_rpc_secret` on all nodes, and raise `garage_replication_factor` (usually to `3`).

> 🖥️ **Web UI:** browse to `http://<garage_webui_bind_host>:3909`. It reaches Garage's admin API and S3 API over the private network, using the admin token the role reads from `secrets/admin_token` (it is written into `.env`, mode 0600).
