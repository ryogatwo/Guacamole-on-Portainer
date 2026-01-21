## Install this Guacamole stack on Raspberry Pi (ARM64)

### 1) Make sure you’re actually on 64-bit ARM

On the Pi:

```bash
uname -m
```

You want `aarch64`. If you’re on 32-bit, switch to a 64-bit OS first (many newer images and Docker releases increasingly assume 64-bit). Docker’s own docs explicitly steer Pi users on 64-bit toward Debian `arm64` packages. ([Docker Documentation][1])

---

### 2) Install Docker (if you don’t already have it)

Use the official Docker docs route (recommended), or the convenience script (fine for lab/testing; Docker notes limitations). ([Docker Documentation][1])

Convenience script (fast path):

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

Then confirm:

```bash
docker --version
docker compose version
sudo docker run --rm hello-world
```

---

### 3) Confirm the images you plan to use exist for ARM64

On the Pi, try pulling your intended images:

```bash
docker pull postgres:16
docker pull guacamole/guacd:latest
docker pull guacamole/guacamole:latest
```

If **either Guacamole pull fails** with something like *“no matching manifest for linux/arm64/v8”*, switch to known ARM64 builds:

* `noloknolo/guacamole` is specifically built for **arm64 + amd64** and is meant to be used the same way as the official image. ([Docker Hub][2])
* `noloknolo/guacd` provides an **arm64** build of guacd. ([Docker Hub][3])

In your Portainer Stack env vars, that would look like:

* `GUACAMOLE_IMAGE=noloknolo/guacamole:latest`
* `GUACD_IMAGE=noloknolo/guacd:1.5.5` (or another tag you prefer)

---

### 4) Create the required host schema file on the Pi

Your compose expects:

`/opt/guacamole/initdb/initdb.sql`

Generate it **on the Pi** using the same Guacamole image you’ll run:

```bash
sudo mkdir -p /opt/guacamole/initdb
sudo docker run --rm ${GUACAMOLE_IMAGE:-guacamole/guacamole:latest} \
  /opt/guacamole/bin/initdb.sh --postgresql \
  | sudo tee /opt/guacamole/initdb/initdb.sql >/dev/null

ls -lh /opt/guacamole/initdb/initdb.sql
```

(That SQL creates the default admin user `guacadmin` / `guacadmin`.) ([Apache Guacamole][4])

---

### 5) Deploy in Portainer

* Portainer → **Stacks** → **Add stack**
* Paste your working `docker-compose.yml`
* Set env vars (minimum: `POSTGRES_PASSWORD`; plus `GUACAMOLE_HTTP_PORT=8081` if you need to avoid conflicts)
* Deploy

Guacamole’s official Docker docs describe the standard 3-container layout (guacamole + guacd + DB), which matches your stack. ([Apache Guacamole][5])

---

### 6) Access & login

Browse to:

`http://<pi-ip>:<GUACAMOLE_HTTP_PORT>/guacamole/`

Login: `guacadmin` / `guacadmin` (then change it). ([Apache Guacamole][4])

---

## One Pi-specific practical note

If your Pi boots from an SD card, Postgres writes can be rough on it. If this will run “for real”, an SSD (USB 3) is kinder to your storage and usually faster.

[1]: https://docs.docker.com/engine/install/raspberry-pi-os/ "Raspberry Pi OS (32-bit / armhf) | Docker Docs"
[2]: https://hub.docker.com/r/noloknolo/guacamole?utm_source=chatgpt.com "noloknolo/guacamole - Docker Image"
[3]: https://hub.docker.com/layers/noloknolo/guacd/1.5.5/images/sha256-0368eb27786e9ae5fa8aad86277d15ee4ac72d16d629763b4fc24119bc86a20d?utm_source=chatgpt.com "Image Layer Details - noloknolo/guacd:1.5.5"
[4]: https://guacamole.apache.org/doc/1.6.0/gug/postgresql-auth.html?utm_source=chatgpt.com "Database setup for PostgreSQL - Apache Guacamole®"
[5]: https://guacamole.apache.org/doc/gug/guacamole-docker.html?utm_source=chatgpt.com "Installing Guacamole with Docker"
