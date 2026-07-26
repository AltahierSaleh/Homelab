# 07 — Auth / SSO

A login wall in front of anything you expose to the internet. Instead of trusting each app's own login, the reverse proxy checks with a central auth server first — one set of credentials, MFA, and a single place to cut access. This is the layer that lets you sleep at night if you port-forward anything.

**Runs on:** an LXC (1 core / 1GB RAM / 8GB disk), Docker + nesting — the standard [LXC baseline](../00-proxmox-host/#3--the-lxc-baseline-used-by-every-container-service).

This lab considered **two options** — pick one:

| | **Authelia** | **Authentik** |
|---|---|---|
| Style | Lightweight, config-file driven | Full identity provider with a web UI |
| RAM | ~150MB idle | ~500MB–1GB (has Postgres + Redis + worker) |
| Setup | Edit YAML | Click through admin UI |
| Protocols | Forward-auth, OIDC | Forward-auth, OIDC, SAML, LDAP |
| Best when | You want minimal + declarative | You want a proper IdP / more apps / SSO everywhere |

If you just want to gate a few web apps behind NPM, **Authelia** is plenty. If you're going to run lots of apps and want real SSO/SAML/LDAP later, **Authentik** scales further.

Both sit **behind NPM** and protect services via **forward-auth**: NPM asks the auth server "is this request logged in?" before passing it through.

---

## Option A — Authelia (lightweight)

```
07-auth/authelia/
├── docker-compose.yml
├── configuration.yml
└── users_database.yml
```

**docker-compose.yml**
```yaml
services:
  authelia:
    image: authelia/authelia:latest
    container_name: authelia
    restart: unless-stopped
    volumes:
      - ./configuration.yml:/config/configuration.yml
      - ./users_database.yml:/config/users_database.yml
    ports:
      - 9091:9091
    environment:
      - TZ=${TZ:-Europe/London}
```

**configuration.yml** (trimmed — see <https://www.authelia.com/configuration/prologue/introduction/> for the full reference):
```yaml
server:
  address: "tcp://0.0.0.0:9091"

# Generate secrets with: `openssl rand -hex 32`
identity_validation:
  reset_password:
    jwt_secret: CHANGE_ME_JWT_SECRET

authentication_backend:
  file:
    path: /config/users_database.yml

access_control:
  default_policy: deny
  rules:
    - domain: "*.<yourdomain>"
      policy: two_factor          # require MFA for everything

session:
  secret: CHANGE_ME_SESSION_SECRET
  cookies:
    - domain: <yourdomain>
      authelia_url: https://auth.<yourdomain>

storage:
  encryption_key: CHANGE_ME_STORAGE_KEY_MIN_20_CHARS
  local:
    path: /config/db.sqlite3

notifier:
  filesystem:
    filename: /config/notification.txt   # dev only; use SMTP in production
```

**users_database.yml** — hash a password with `docker run --rm authelia/authelia:latest authelia crypto hash generate argon2 --password 'yourpassword'`:
```yaml
users:
  <yourusername>:
    displayname: "Your Name"
    password: "$argon2id$v=19$...PASTE_HASH..."
    email: you@example.com
    groups:
      - admins
```

Launch: `docker compose up -d`.

---

## Option B — Authentik (full IdP)

Authentik ships an official compose file — don't hand-roll it:

```bash
sudo mkdir -p /opt/authentik && cd /opt/authentik
wget https://goauthentik.io/docker-compose.yml
```

Create a `.env` next to it:
```env
PG_PASS=CHANGE_ME            # openssl rand -base64 36 | tr -d '\n'
AUTHENTIK_SECRET_KEY=CHANGE_ME   # openssl rand -base64 60 | tr -d '\n'
# Optional email:
# AUTHENTIK_EMAIL__HOST=smtp.example.com
```

Launch and finish setup in the browser:
```bash
docker compose pull
docker compose up -d
```
Go to `http://<LXC-IP>:9000/if/flow/initial-setup/` to create the admin (akadmin) account.

In the Authentik UI you then create a **Proxy Provider** + **Application** per service, and add an **Outpost** that NPM forwards to. Full walkthrough: <https://docs.goauthentik.io/>.

---

## Wiring it to Nginx Proxy Manager (either option)

Both integrate the same way — NPM does a **forward-auth** subrequest to the auth server, and unauthenticated users get bounced to the login page.

1. Add a proxy host in NPM for the auth server itself: `auth.<yourdomain>` → the auth LXC (`:9091` Authelia / `:9000` Authentik).
2. On each **protected** proxy host, open the **Advanced** tab and add an auth snippet. For Authelia it looks like:
   ```nginx
   location /authelia {
       internal;
       set $upstream_authelia http://<AUTH_LXC_IP>:9091/api/verify;
       proxy_pass $upstream_authelia;
       proxy_set_header X-Original-URL $scheme://$http_host$request_uri;
   }
   location / {
       auth_request /authelia;
       auth_request_set $target_url $scheme://$http_host$request_uri;
       error_page 401 =302 https://auth.<yourdomain>/?rd=$target_url;
       # ... then your normal proxy_pass to the app
   }
   ```
   Authentik provides its own snippet from the Outpost setup screen — paste that instead.
3. Set `default_policy: deny` (Authelia) so nothing is exposed unless you explicitly allow it.

> Rule of thumb: put **everything internet-facing** behind auth. Purely-LAN services (only reachable over your VPN or local network) can skip it if you prefer convenience.

---

## Done

That's the full lab. Loop back to the **[root README](../README.md)** for the architecture overview, or revisit any layer. From here, good next steps: automated backups of every `config` volume, and a scheduled `docker compose pull` to keep images current.
