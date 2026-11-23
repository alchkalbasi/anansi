# Vaultwarden Deployment Stack

An **Ansible role** paired with a **Docker Compose** stack that deploys a self-hosted company password manager with [Vaultwarden](https://github.com/dani-garcia/vaultwarden). Vaultwarden is a Bitwarden-compatible server, so employees use the **official Bitwarden** web vault, browser extensions, desktop apps and mobile apps.

The role checks the target host, prepares directories, generates and keeps secrets, templates the settings, starts Vaultwarden with PostgreSQL behind Traefik, and installs a nightly backup cron job.

---

## Prerequisites

* **Operating system:** Linux (Ubuntu/Debian recommended)
* **On the host:** `docker-ce` and `docker-compose-plugin` (see the `docker` role)
* **On the control node:** the `community.docker` collection
* **HTTPS:** a DNS name pointing at the host, and the `traefik` role (or another TLS reverse proxy). The web vault and the browser extensions refuse to work over plain HTTP.
* **SMTP (strongly recommended):** needed for invitation emails, email verification and 2FA by email.

---

## Layout on the host

```
/opt/services/vaultwarden/
├── compose.yml
├── .env                  # compose variables (0600)
├── vaultwarden.env       # Vaultwarden settings and secrets (0600)
├── data/                 # RSA keys, attachments, sends, icon cache
├── postgres/             # PostgreSQL data
├── backups/<timestamp>/  # db.sql.gz + data.tar.gz
└── secrets/              # 0700
    ├── postgres_password
    └── admin_token       # login token for /admin
```

---

## Variables

The main ones are below. See [`defaults/main.yml`](defaults/main.yml) for the full list.

| Variable | Default | Description |
| --- | --- | --- |
| `vaultwarden_domain` | `https://vault.example.com` | Public URL. Must be `https://` |
| `vaultwarden_image_tag` | `1.37.5` | `vaultwarden/server` image tag |
| `vaultwarden_bind_host` / `vaultwarden_http_port` | `127.0.0.1` / `8222` | Direct HTTP port, used by local health checks or a non-Traefik proxy |
| `vaultwarden_traefik_enabled` | `true` | Add Traefik labels (entrypoint `web-secure`, resolver `le`) |
| `vaultwarden_admin_allowed_ips` | private ranges | CIDRs that may open `/admin` through Traefik |
| `vaultwarden_admin_enabled` | `true` | Turn on the `/admin` panel |
| `vaultwarden_admin_token` | `""` | Empty means a random token is generated once. Better: an Argon2 hash (see below) |
| `vaultwarden_postgres_password` | `""` | Empty means a random one is generated once |
| `vaultwarden_signups_allowed` | `false` | Open registration. Keep `false`: invite only |
| `vaultwarden_signups_domains_whitelist` | `[]` | Email domains that may self-register even with signups closed |
| `vaultwarden_org_creation_users` | `[]` | Emails allowed to create organizations (empty means everyone) |
| `vaultwarden_org_groups_enabled` | `true` | Groups (teams). Upstream still marks this as beta |
| `vaultwarden_experimental_custom_roles` | `false` | The **Custom** member role with fine-grained permissions (experimental) |
| `vaultwarden_org_events_enabled` | `true` | Audit event log per organization |
| `vaultwarden_emergency_access_allowed` | `false` | Personal emergency access. Off by default for a company vault |
| `vaultwarden_smtp_*` | empty | SMTP server for emails |
| `vaultwarden_sso_*` | disabled | OpenID Connect SSO (Keycloak, Authentik, Entra ID…) |
| `vaultwarden_push_*` | disabled | Mobile push sync through Bitwarden's relay (get an id/key at <https://bitwarden.com/host>) |
| `vaultwarden_extra_env` | `{}` | Any other [Vaultwarden setting](https://github.com/dani-garcia/vaultwarden/blob/main/.env.template) |
| `vaultwarden_backup_enabled` | `true` | Nightly `pg_dump` and data archive, kept for `vaultwarden_backup_retention_days` (14) |

Example (keep the secrets in ansible-vault):

```yaml
vaultwarden_domain: https://vault.company.com
vaultwarden_org_creation_users: [it-admin@company.com]
vaultwarden_invitation_org_name: "Company Vault"
vaultwarden_admin_token: "{{ vault_vaultwarden_admin_token }}" # Argon2 PHC string
vaultwarden_smtp_host: smtp.company.com
vaultwarden_smtp_from: vault@company.com
vaultwarden_smtp_username: vault@company.com
vaultwarden_smtp_password: "{{ vault_vaultwarden_smtp_password }}"
```

---

## Usage

```bash
ansible-playbook -i inventory.ini playbook.yml --tags vaultwarden
```

| Tag | What it does |
| --- | --- |
| `check-docker` | Checks that Docker and Compose are installed |
| `prepare` | Checks the variables, creates directories and copies `compose.yml` |
| `configure` | Writes secrets, `.env` and `vaultwarden.env` |
| `deploy` | Creates the networks, pulls the images, runs `docker compose up` and waits for health |
| `backup` | Installs the backup script and cron job |

---

## First-time setup

1. **Open the admin panel** at `https://<domain>/admin` from an allowed IP. The token is in `sudo cat /opt/services/vaultwarden/secrets/admin_token`.
2. **Invite the first admin** (Users → Invite User), accept the invite and create an account.
3. **Create the organization** in the web vault (New organization).
4. **Switch the admin token to an Argon2 hash** (recommended):
   ```bash
   docker exec -it vaultwarden /vaultwarden hash --preset owasp
   ```
   Put the `$argon2id$...` output in `vaultwarden_admin_token` (ansible-vault) and run the role again. From then on you log in to `/admin` with the password you typed, not the hash.

> ⚠️ When you click **Save** in the `/admin` settings page, Vaultwarden writes `data/config.json`, and that file **overrides** the environment values from this role. Manage settings through Ansible, or delete `config.json` if the two drift apart.

---

## RBAC and PBAC: how to model your company

All of this is set up in the web vault under the organization (**Admin Console**).

### Roles (RBAC)

| Role | Can do |
| --- | --- |
| **Owner** | Everything, including billing-style settings, deleting the org and managing other owners |
| **Admin** | Manage members, groups, collections and policies. Sees all items |
| **Manager** (legacy) / **User** with *Manage collection* | Manage only the collections assigned to them |
| **User** | Use only the collections assigned to them, with the permission given there |
| **Custom** | Fine-grained permissions (only with `vaultwarden_experimental_custom_roles: true`) |

### Groups = teams

With `vaultwarden_org_groups_enabled: true`, create groups such as `devops`, `finance` or `support`, add members, and give **collections** to groups rather than to people. When someone joins or leaves a team, you change one group membership.

### Collection permissions (the "autofill without seeing" part)

For each collection, a member or group gets one of:

| Permission | Effect |
| --- | --- |
| **Can view** | Read and autofill |
| **Can view, except passwords** | Autofill works, but the password (and hidden fields) are masked and cannot be revealed or copied |
| **Can edit** | Create and change items |
| **Can edit, except passwords** | Edit items but not see the passwords |
| **Can manage** | Edit plus manage who has access to the collection |

Typical layout: a `shared-tools` collection given to most groups with **Can view, except passwords**, and the same collection given to the `it-admins` group with **Can manage**.

> 🔐 **Know the limit:** "except passwords" is enforced by the Bitwarden **client**, not by cryptography. The client must decrypt the password to autofill it, so a technical user can still get it out (browser dev tools, autofilling into a page they control). It stops casual viewing and copying. It does not protect against a determined insider. For high-value secrets, do not give access at all, and rotate passwords when people leave.

### Policies (PBAC)

Under **Admin Console → Settings → Policies**, enable at least:

* **Require two-step login** for all members
* **Master password requirements** (minimum length and complexity)
* **Password generator** (minimum rules for generated passwords)
* **Single organization**, so members cannot join other orgs on this server
* **Remove individual vault**, so company items always go into the organization
* **Account recovery administration**, so admins can reset a member's master password (members must enrol)
* **Remove Send / Send options**, if sharing through Send is not wanted

### Audit

With `vaultwarden_org_events_enabled: true`, **Admin Console → Reporting → Event logs** shows logins, item views, autofills, password copies and permission changes, kept for `vaultwarden_events_days_retain` days.

---

## Clients

Employees install the official **Bitwarden** browser extension, desktop app or mobile app. On the login screen, choose **Self-hosted** and enter `vaultwarden_domain`.

---

## Backups and restore

The cron job (`/etc/cron.d/vaultwarden-backup`, 02:30 every night) writes `backups/<timestamp>/db.sql.gz` and `data.tar.gz` and logs to `/var/log/vaultwarden-backup.log`. Run it by hand with `sudo /usr/local/bin/vaultwarden-backup.sh`.

> 💾 The backups stay on the same host. Copy `backups/` off the server (rsync, restic to the `garage` role, etc.). Losing the password vault is much worse than losing most services.

Restore:

```bash
cd /opt/services/vaultwarden
docker compose stop vaultwarden
gunzip -c backups/<ts>/db.sql.gz | docker exec -i vaultwarden-db psql -U vaultwarden -d vaultwarden
rm -rf data && tar -xzf backups/<ts>/data.tar.gz
docker compose start vaultwarden
```

---

## After deployment

```bash
docker compose -f /opt/services/vaultwarden/compose.yml ps
curl -fsS http://127.0.0.1:8222/alive
docker logs -f vaultwarden
```
