# Deploy a Spring Boot JAR the “Linux-native” way using FHS + systemd + optional Nginx.

---

## 1. What is the **FHS (Filesystem Hierarchy Standard)?**

It’s a set of conventions that says *where* things should live on a Linux system.
Why? So that all apps, tools, and sysadmins know where to look for stuff — no surprises.

Example:

* Logs usually live in `/var/log/...` → so `logrotate`, `journalctl`, monitoring tools, etc. can find them.
* Configs go in `/etc/...` → so sysadmins know where to tweak settings.
* Binaries in `/usr/bin`, `/opt/...` → so they don’t clutter random places.

Think of it like: **a tidy house where the kitchen, bathroom, and bedroom are always in the same place.**

---

## 2. How does this apply to *any* app?

Yes — this is the **standard** for *all Linux apps*.

* Apache, Nginx, MySQL, Postgres… they all follow this structure.
* If you build your app this way, sysadmins treat it just like those apps.

It’s not “mandatory” (your app still runs if you just `java -jar myapp.jar` from your home folder), but following FHS makes it more **professional and maintainable**.

---

## 3. How does this relate to a **Spring Boot app**?

Spring Boot gives you a “fat jar” (executable JAR) that can run anywhere with `java -jar`.
That’s great for development, but in production you don’t want everything in one random folder. Instead:

* **Executable JAR**
  Put it in `/opt/myapp/myapp.jar`
  → `/opt` is where “add-on software” lives (things not part of the base system).

* **Configs** (`application.yml`, `.env`)
  Put in `/etc/myapp/`
  → This way, when you redeploy a new JAR version, your configs survive.

* **Logs**
  Go to `/var/log/myapp/`
  → Or you let `systemd` handle logs (view with `journalctl -u myapp`).

* **Persistent data** (uploads, caches, DB files if embedded)
  Put in `/var/lib/myapp/`
  → That’s the “stateful data” place.

* **Service management**
  Create a `systemd` service file (`/etc/systemd/system/myapp.service`) so you can do:

  ```bash
  sudo systemctl start myapp
  sudo systemctl enable myapp
  sudo systemctl status myapp
  ```

  → This makes it behave like any system service.

* **Reverse proxy / TLS**
  If you use Nginx, config goes in `/etc/nginx/sites-available/` (with a symlink into `sites-enabled/`).

* **Log rotation**
  If you write logs to files, put a config in `/etc/logrotate.d/myapp` → so old logs get rotated, compressed, and deleted automatically.

---

## 4. Beginner-friendly analogy

Imagine your Spring Boot JAR is a **toy robot** 🦾:

* By default, you just dump it on your desk (your dev machine) and it runs.
* Following FHS is like putting it into a **toolbox with labeled drawers** in a workshop:

    * `/opt/myapp/` → where the robot itself lives.
    * `/etc/myapp/` → where you keep the robot’s instructions (configs).
    * `/var/log/myapp/` → where you store the robot’s diary (logs).
    * `/var/lib/myapp/` → where the robot stores its memory (persistent data).
    * `systemd` → the workshop switch to turn the robot on/off.
    * `nginx` → the receptionist who speaks to visitors and forwards requests to your robot.

This way, any mechanic (sysadmin) walking in knows exactly where to look — they don’t need to guess.

---



```java





```

# Step-by-step guide to deploy a Spring Boot JAR the “Linux-native” way

---

## 0) Assumptions

* App name: `myapp`
* JAR file you built: `target/myapp-1.0.0.jar`
* App listens on port `8080` (default Spring Boot)

You can change names/ports—just keep paths consistent.

---

## 1) Create a dedicated user and folders

```bash
# 1) Least-privilege system user (no login shell)
sudo useradd --system --home /opt/myapp --shell /usr/sbin/nologin myapp

# 2) FHS directories
sudo mkdir -p /opt/myapp
sudo mkdir -p /etc/myapp
sudo mkdir -p /var/log/myapp
sudo mkdir -p /var/lib/myapp

# 3) Permissions (only myapp can read/write its stuff)
sudo chown -R myapp:myapp /opt/myapp /var/log/myapp /var/lib/myapp
sudo chown root:root /etc/myapp
sudo chmod 750 /opt/myapp /var/log/myapp /var/lib/myapp
sudo chmod 750 /etc/myapp
```

---

## 2) Install the JAR

```bash
# Copy your built jar into /opt/myapp and standardize the name
sudo cp target/myapp-1.0.0.jar /opt/myapp/myapp.jar
sudo chown myapp:myapp /opt/myapp/myapp.jar
sudo chmod 640 /opt/myapp/myapp.jar
```

Tip: for upgrades, replace `myapp.jar` atomically (see “Upgrades” below).

---

## 3) Externalize configuration

### 3.1 `/etc/myapp/application.yml`

```yaml
server:
  port: 8080

spring:
  profiles:
    active: prod

# If you want file logging (optional, see Section 6)
# logging:
#   file:
#     name: /var/log/myapp/app.log
```

```bash
sudo nano /etc/myapp/application.yml
sudo chown root:root /etc/myapp/application.yml
sudo chmod 640 /etc/myapp/application.yml
```

### 3.2 `/etc/myapp/myapp.env` (environment variables)

```ini
# Example secrets and tuning (do NOT commit this to git)
JAVA_OPTS="-Xms256m -Xmx512m"
SPRING_DATASOURCE_URL="jdbc:postgresql://db:5432/mydb"
SPRING_DATASOURCE_USERNAME="myuser"
SPRING_DATASOURCE_PASSWORD="supersecret"
# If you use a random secret:
# SPRING_SECURITY_OAUTH2_CLIENT_REGISTRATION_...=...
```

```bash
sudo nano /etc/myapp/myapp.env
sudo chown root:root /etc/myapp/myapp.env
sudo chmod 640 /etc/myapp/myapp.env
```

> Spring Boot will read env vars automatically (`SPRING_*`). We’ll also pass `--spring.config.additional-location` so it loads `/etc/myapp/application.yml`.

---

## 4) Create a `systemd` service

`/etc/systemd/system/myapp.service`:

```ini
[Unit]
Description=MyApp Spring Boot Service
After=network.target

[Service]
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
EnvironmentFile=/etc/myapp/myapp.env
# Load external config from /etc plus allow env vars to override
ExecStart=/usr/bin/java $JAVA_OPTS \
  -jar /opt/myapp/myapp.jar \
  --spring.config.additional-location=file:/etc/myapp/application.yml

# Restart rules
Restart=always
RestartSec=5

# Security hardening (safe defaults; relax if needed)
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=full
ProtectHome=true
ReadWritePaths=/var/log/myapp /var/lib/myapp
# If you use port <1024, you'll need capabilities or a reverse proxy

# Logging to journald (journalctl -u myapp)
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

Enable + start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable myapp
sudo systemctl start myapp
sudo systemctl status myapp
# View logs
journalctl -u myapp -f
```

---

## 5) (Optional) Put Nginx in front (reverse proxy + TLS)

`/etc/nginx/sites-available/myapp.conf`:

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass         http://127.0.0.1:8080;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_read_timeout 60s;
    }
}
```

Enable and test:

```bash
sudo ln -s /etc/nginx/sites-available/myapp.conf /etc/nginx/sites-enabled/myapp.conf
sudo nginx -t
sudo systemctl reload nginx
```

TLS (recommended): use certbot/Let’s Encrypt to upgrade the server block to `listen 443 ssl` with real certs.

---

## 6) Choose your logging strategy

### A) **Use journald only** (simplest)

* Already configured in the service.
* Tail logs: `journalctl -u myapp -f`
* No logrotate needed (journald handles rotation).

### B) **Write to files** in `/var/log/myapp/` (if you must)

1. Enable file logging (uncomment in `application.yml` or use env vars):

```yaml
logging:
  file:
    name: /var/log/myapp/app.log
```

2. Logrotate config `/etc/logrotate.d/myapp`:

```conf
/var/log/myapp/*.log {
  daily
  rotate 14
  compress
  missingok
  notifempty
  copytruncate
}
```

3. Permissions:

```bash
sudo chown myapp:myapp /var/log/myapp
sudo chmod 750 /var/log/myapp
```

---

## 7) App data (persistent state)

Put any uploads/caches/db files under:

```
/var/lib/myapp/
```

Make sure your app points there (via config/env). The systemd unit allows write access to this path.

---

## 8) Health check & firewall

```bash
# If your app exposes /actuator/health
curl -i http://127.0.0.1:8080/actuator/health

# Optional firewall (UFW example—only expose 80/443 if using Nginx)
sudo ufw allow 80
sudo ufw allow 443
# Keep 8080 internal only (don’t allow from the internet)
```

---

## 9) Upgrades (zero-ish downtime)

```bash
# 1) Copy new jar alongside (atomic swap)
sudo cp target/myapp-1.1.0.jar /opt/myapp/myapp.jar.new
sudo chown myapp:myapp /opt/myapp/myapp.jar.new
sudo chmod 640 /opt/myapp/myapp.jar.new

# 2) Swap
sudo mv /opt/myapp/myapp.jar.new /opt/myapp/myapp.jar

# 3) Restart service
sudo systemctl restart myapp
sudo systemctl status myapp
```

> Configs in `/etc/myapp` and data in `/var/lib/myapp` remain untouched.

---

## 10) Uninstall (clean removal)

```bash
sudo systemctl stop myapp
sudo systemctl disable myapp
sudo rm -f /etc/systemd/system/myapp.service
sudo systemctl daemon-reload

# Optional: remove files (careful: data/logs!)
# sudo rm -rf /opt/myapp /etc/myapp /var/lib/myapp /var/log/myapp
# sudo userdel myapp
```

---

## Quick mental model (recap)

* `/opt/myapp` → the **program** (jar)
* `/etc/myapp` → **settings/secrets**
* `/var/log/myapp` → **logs**
* `/var/lib/myapp` → **data**
* `systemd` → **start/stop/restart & autostart**
* `nginx` → **public entry + TLS**


















