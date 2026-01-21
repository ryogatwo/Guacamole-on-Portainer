

# README — Apache Guacamole on Portainer (Ubuntu x64)

## What this deploys

This Portainer stack deploys:

* `guacamole/guacamole` (web app on port 8080 inside the container)
* `guacamole/guacd` (proxy daemon)
* `postgres:16` (authentication/config DB)

It uses a **host-generated schema file** to initialize the DB on first start:

* Host file (required): `/opt/guacamole/initdb/initdb.sql`
* Mounted to Postgres: `/docker-entrypoint-initdb.d/initdb.sql`

**Important:** Postgres only runs init scripts when the DB volume is new/empty.

---

## Prerequisites

* Ubuntu x64 host running Docker
* Portainer running and managing this Docker environment
* DNS name `linuxdocker.corp` (or use the host IP)

---

## Step 1 — Create the required schema file on the Docker host

Run this on the Ubuntu host:

```bash
sudo mkdir -p /opt/guacamole/initdb && sudo docker run --rm guacamole/guacamole:latest /opt/guacamole/bin/initdb.sh --postgresql | sudo tee /opt/guacamole/initdb/initdb.sql >/dev/null && ls -lh /opt/guacamole/initdb/initdb.sql
```

Expected result: you should see `initdb.sql` created (about ~24 KB).

---

## Step 2 — Create the stack in Portainer

1. Portainer → **Stacks** → **Add stack**
2. Name it something like: `guacamole`
3. Paste your `docker-compose.yml` (the one with explicit comments)

---

## Step 3 — Add Stack environment variables in Portainer

In the same stack screen, add these **Environment variables**:

### Required

* `POSTGRES_PASSWORD` = **set a strong password**

### Recommended

* `GUACAMOLE_HTTP_PORT` = `8080`

  * If port 8080 is already used, use `8081`, `8082`, etc.
* `POSTGRES_DB` = `guacamole_db`
* `POSTGRES_USER` = `guacamole_user`

### Optional (pin images)

* `POSTGRES_IMAGE` = `postgres:16`
* `GUACAMOLE_IMAGE` = `guacamole/guacamole:latest`
* `GUACD_IMAGE` = `guacamole/guacd:latest`

---

## Step 4 — Deploy the stack

Click **Deploy the stack**.

---

## Step 5 — Open Guacamole and log in

Open:

* `http://linuxdocker.corp:<GUACAMOLE_HTTP_PORT>/guacamole/`

Default login created by the schema:

* Username: `guacadmin`
* Password: `guacadmin`

**Immediately change the password:**

* Guacamole → **Settings** → **Users** → **guacadmin** → **Change Password**

---

## Verification commands (Ubuntu host)

Check containers:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}" | egrep "guacamole|postgres|guacd"
```

Confirm schema tables exist:

```bash
docker exec -it $(docker ps --format "{{.Names}}" | grep -i postgres | head -n 1) \
  psql -U guacamole_user -d guacamole_db -c "\dt" | head -n 20
```

You should see `guacamole_*` tables.

---

## Common gotchas and fixes

### 1) “Error has occurred…” on the web UI

This usually means Guacamole can’t query the DB.

**Check Postgres is healthy:**

```bash
docker ps --format "table {{.Names}}\t{{.Status}}" | grep -i postgres
```

**If Postgres is healthy but tables are missing**, you reused an old DB volume.

Fix: remove the stack and its volume, then redeploy (warning: deletes DB data):

```bash
docker compose -p guacamole down -v
```

Or in Portainer: remove stack → remove the stack’s Postgres volume → deploy again.

### 2) Schema didn’t initialize

Postgres only runs `/docker-entrypoint-initdb.d/*.sql` when the data directory is empty.

Fix: delete the `guac_db` volume and redeploy (same warning: data loss).

### 3) Port already in use

Change `GUACAMOLE_HTTP_PORT` to a free port (8081, 8082, …).

---

## Removing a stack cleanly

Portainer:

* **Stacks → (stack) → Remove** (optionally remove volumes)

CLI:

```bash
docker compose -p guacamole down -v
```

---

## Reusing as a Portainer Custom Template

Once deployed and working:

1. Portainer → **Stacks** → click the stack
2. Click **Create template from stack**
3. Save it as a Custom Template for future deployments

---



### One practical follow-up

If you ever see login weirdness, check DB health first:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}" | egrep "guacamole|postgres|guacd"
```

If `postgres` is **healthy** and Guacamole is **up**, a `docker restart guacamole-guacamole-1` is a safe first move.



