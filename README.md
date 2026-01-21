# Copyparty Deployment (Debian / Ubuntu)

A **clean, no‑nonsense, production‑ready** guide to deploying **Copyparty** on Debian/Ubuntu using the **official SFX (`copyparty-sfx.py`) approach**, with:

* systemd service
* Apache reverse proxy (HTTPS)
* external authentication via headers / DB auth
* sane filesystem layout

This repo is meant to be **copied, audited, and adapted**, not blindly run.

---

## What this repo is (and is not)

**This repo IS:**

* a documented reference deployment
* suitable for VPS / homelab / small production
* explicit about security boundaries

**This repo IS NOT:**

* a Docker setup
* a Copyparty fork
* a one‑click installer

If you want Docker or Kubernetes, this is not for you.

---

## Tested environment

* Debian 12 / Ubuntu 22.04+
* Python 3.10+
* Apache 2.4
* Copyparty SFX (latest release)

---

## Directory layout (host system)

These paths are deliberate. Don’t freestyle unless you know why.

```
/opt/copyparty/
└── copyparty-sfx.py        # upstream binary (no edits)

/srv/copyparty/
├── cfg/
│   └── copyparty.conf     # main config
├── users/
│   └── <username>/        # per-user storage roots
└── logs/                  # optional, if you don’t rely on journald
```

---

## System prerequisites

### Core packages

```bash
sudo apt update
sudo apt install -y \
  python3 \
  ca-certificates \
  curl wget
```

### Recommended (thumbnails + metadata)

Copyparty explicitly recommends these on Debian:

```bash
sudo apt install -y \
  python3-pip \
  python3-dev \
  ffmpeg
```

Install Pillow **as the Copyparty runtime user**, not root:

```bash
python3 -m pip install --user -U Pillow pillow-avif-plugin
```

---

## Create service user

Never run Copyparty as root. Period.

```bash
sudo useradd -r -s /usr/sbin/nologin copyparty
```

---

## Install Copyparty (SFX)

Create directories:

```bash
sudo mkdir -p /opt/copyparty /srv/copyparty/{cfg,users}
sudo chown -R copyparty:copyparty /opt/copyparty /srv/copyparty
```

Download the **latest `copyparty-sfx.py`** from upstream:

```bash
cd /opt/copyparty
# open releases page, copy the direct SFX asset URL
python3 -c 'import webbrowser; webbrowser.open("https://github.com/9001/copyparty/releases/latest")'

sudo wget -O copyparty-sfx.py "PASTE_DIRECT_URL_HERE"
sudo chmod 755 copyparty-sfx.py
```

Quick sanity test (local only):

```bash
sudo -u copyparty python3 /opt/copyparty/copyparty-sfx.py --http 3923
```

If this doesn’t start cleanly, **stop here** and fix it.

---

## Copyparty configuration

Location:

```
/srv/copyparty/cfg/copyparty.conf
```

Minimal, sane config matching a reverse‑proxy setup:

```ini
[global]
  idp-h-usr: X-Authooley-User
  xff-src: 127.0.0.1
  shr: /shr

[/u/${u}]
  /srv/copyparty/users/${u}
  accs:
    rwmda: ${u}
```

**What this does:**

* trusts auth headers from localhost only
* maps users to `/srv/copyparty/users/<user>`
* enables full access for authenticated users

---

## systemd service

Create:

```
/etc/systemd/system/copyparty.service
```

```ini
[Unit]
Description=Copyparty File Server
After=network.target

[Service]
Type=simple
User=copyparty
Group=copyparty
WorkingDirectory=/srv/copyparty

ExecStart=/usr/bin/python3 /opt/copyparty/copyparty-sfx.py \
  --theme=1 \
  --rproxy=1 \
  --xff-hdr=X-Forwarded-For \
  -c /srv/copyparty/cfg/copyparty.conf

Restart=on-failure
RestartSec=3
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now copyparty
sudo journalctl -u copyparty -f
```

---

## Apache reverse proxy (HTTPS)

### Install Apache + SSL

```bash
sudo apt install -y apache2 certbot python3-certbot-apache
sudo a2enmod proxy proxy_http headers ssl
```

### VirtualHost example

```
/etc/apache2/sites-available/copyparty.conf
```

```apache
<VirtualHost *:443>
    ServerName bfs.example.net

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/bfs.example.net/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/bfs.example.net/privkey.pem

    ProxyPreserveHost On
    ProxyPass / http://127.0.0.1:3923/
    ProxyPassReverse / http://127.0.0.1:3923/

    RequestHeader set X-Forwarded-Proto "https"

    # Auth handled here (DB / LDAP / SSO)
    RequestHeader set X-Authooley-User "%{REMOTE_USER}s"
    RequestHeader unset Authorization
</VirtualHost>
```

Enable site + cert:

```bash
sudo a2ensite copyparty
sudo systemctl reload apache2
sudo certbot --apache -d bfs.example.net
```

---

## Authentication model (important)

Copyparty **does not authenticate users itself** here.

Instead:

```
[ Browser ] → [ Apache Auth ] → [ Header ] → [ Copyparty ]
```

Apache is the **security boundary**.
Copyparty trusts `X-Authooley-User` **only from localhost**.

If Apache auth is broken, your instance is broken.

---

## Permissions & storage

Each user gets:

```
/srv/copyparty/users/<username>
```

Ownership:

```bash
sudo chown -R copyparty:copyparty /srv/copyparty/users
```

Do **not** expose arbitrary filesystem paths unless you understand the risk.

---

## Upgrading Copyparty

1. Stop service
2. Replace `copyparty-sfx.py`
3. Start service

```bash
sudo systemctl stop copyparty
sudo wget -O /opt/copyparty/copyparty-sfx.py NEW_URL
sudo systemctl start copyparty
```

Configs and data remain untouched.

---

## Logs & debugging

* Copyparty: `journalctl -u copyparty`
* Apache: `/var/log/apache2/*`
* Reverse‑proxy issues = Apache logs
* Permission issues = filesystem ownership

Most problems are **not Copyparty bugs** — they’re proxy or auth mistakes.

---

## Security notes (read this)

* ❌ Never commit DB passwords or secrets
* ✅ Use placeholders in configs
* ✅ Run as unprivileged user
* ✅ TLS termination at proxy only
* ❌ Do not expose port 3923 publicly

If you expose Copyparty directly to the internet without a proxy, that’s on you.

---

## Why this setup

* SFX = zero Python dependency hell
* systemd = predictable lifecycle
* Apache = mature auth + TLS
* Header auth = SSO‑friendly

It’s boring — and boring is good.

---

## License & upstream

* Copyparty © upstream authors
* This repo contains **no Copyparty code**, only deployment examples

Upstream project:
[https://github.com/9001/copyparty](https://github.com/9001/copyparty)

---

**If you’re reading this in GitHub and it feels boring and strict — good.**
That means it’s production‑ready.
