# Kutt Installation and Management using Docker

Kutt is an open-source URL shortener designed for simplicity, speed, and flexibility. It allows users to create short links, manage them through a clean web interface, and track analytics such as click counts and referrers. Kutt supports custom domains, password-protected links, API access for automation, and self-hosting for full control over your data. With its sleek design and powerful features, Kutt is a great choice for personal use, teams, or businesses looking for a reliable and privacy-friendly link-shortening solution.

**Target OS:** Ubuntu 24.04 LTS
**Environment:** Development
**Database:** SQLite (default, embedded)

---

## 1. Install Docker

### 1.1 Remove any old Docker versions

```bash
sudo apt remove -y docker docker-engine docker.io containerd runc
```

### 1.2 Install dependencies

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release
```

### 1.3 Add Docker GPG key

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

### 1.4 Add Docker repository

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 1.5 Install Docker Engine and Compose plugin

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 1.6 Test installation

```bash
sudo docker run hello-world
docker compose version
```

### 1.7 Optional: Allow running Docker without sudo

```bash
sudo usermod -aG docker $USER
newgrp docker
```

---

## 2. Clone Kutt Source and Configure

### 2.1 Clone the repository

```bash
git clone https://github.com/thedevs-network/kutt.git
cd kutt
```

### 2.2 Create env file

```bash
cp .example.env .env
```

Edit `.env` and set at minimum:

```
# Optional - App port to run on
PORT=3000

# Optional - The name of the site where Kutt is hosted
SITE_NAME=Kutt

# Optional - The domain that this website is on
# DEFAULT_DOMAIN=example.com
DEFAULT_DOMAIN=localhost


# Required - A passphrase to encrypt JWT. Use a random long string
JWT_SECRET=change_this_to_a_secure_random_value

# Optional - Database client. Available clients for the supported databases:
# pg | better-sqlite3 | mysql2
# other supported drivers that you can use but you have to manually install them with npm:
# pg-native | sqlite3 | mysql
DB_CLIENT=better-sqlite3

# Optional - SQLite database file path
# Only if you're using SQLite
DB_FILENAME=db/data

# Optional - SQL database credential details
# Only if you're using Postgres or MySQL
DB_HOST=localhost
DB_PORT=5432
DB_NAME=kutt
DB_USER=postgres
DB_PASSWORD=
DB_SSL=false
DB_POOL_MIN=0
DB_POOL_MAX=10

# Optional - Generated link length
LINK_LENGTH=6

# Optional - Alphabet used to generate custom addresses
# Default value omits o, O, 0, i, I, l, 1, and j to avoid confusion when reading the URL
LINK_CUSTOM_ALPHABET=abcdefghkmnpqrstuvwxyzABCDEFGHKLMNPQRSTUVWXYZ23456789

# Optional - Tells the app that it's running behind a proxy server
# and that it should get the IP address from that proxy server
# if you're not using a proxy server then set this to false, otherwise users can override their IP address
TRUST_PROXY=true

# Optional - Redis host and port
REDIS_ENABLED=false
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
REDIS_PASSWORD=
# The number for Redis database, between 0 and 15. Defaults to 0.
# If you don't know what this is, then you probably don't need to change it.
REDIS_DB=0

# Optional - Disable registration. Default is true.
DISALLOW_REGISTRATION=true

# Optional - Disable anonymous link creation. Default is true.
DISALLOW_ANONYMOUS_LINKS=true

# Optional - This would be shown to the user on the settings page
# It's only for display purposes and has no other use
SERVER_IP_ADDRESS=
SERVER_CNAME_ADDRESS=

# Optional - Use HTTPS for links with custom domain
# It's on you to generate SSL certificates for those domains manually, at least on this version for now
CUSTOM_DOMAIN_USE_HTTPS=false

# Optional - Email is used to verify or change email address, reset password, and send reports.
# If it's disabled, all the above functionality would be disabled as well.
# MAIL_FROM example: "Kutt <support@kutt.it>". Leave it empty to use MAIL_USER.
# More info on the configuration on http://nodemailer.com/.
MAIL_ENABLED=false
MAIL_HOST=
MAIL_PORT=587
MAIL_SECURE=true
MAIL_USER=
MAIL_FROM=
MAIL_PASSWORD=

# Optional - Enable rate limitting for some API routes
ENABLE_RATE_LIMIT=false

# Optional - The email address that will receive submitted reports
REPORT_EMAIL=

# Optional - Support email to show on the app
CONTACT_EMAIL=
```

Generate a secure secret (optional):

```bash
openssl rand -base64 48
```

SQLite will be used automatically in development mode.

---

## 3. Start Kutt using Docker Compose (SQLite)

Kutt provides a default `docker-compose.yml` suitable for development.

### 3.1 Start containers

```bash
docker compose up -d
```

Services will start as background daemons.

### 3.2 Check status

```bash
docker compose ps
docker compose logs -f
```

### 3.3 First-time access

Open browser:

```
http://localhost:3000
```

Create the admin account when prompted.

---

## 4. Container Management Commands

Stop containers:

```bash
docker compose down
```

Restart containers:

```bash
docker compose restart
```

View logs:

```bash
docker compose logs -f server
```

List running containers:

```bash
docker ps
```

---

## 5. Redeployment Procedure

When updating Kutt to a newer release:

### 5.1 Pull newest image

```bash
docker compose pull
```

### 5.2 Recreate containers with latest image

```bash
docker compose up -d
```

### 5.3 Check that service is healthy

```bash
docker compose ps
docker compose logs -f
```

Database and settings are preserved through volumes.

---

## 6. Enable Autostart on System Reboot

### 6.1 Enable Docker service autostart

```bash
sudo systemctl enable docker
```

### 6.2 Ensure containers start on boot

Add `restart` policy in `docker-compose.yml` under the server service:

```
restart: unless-stopped
```

Example:

```yaml
services:
  server:
    build:
      context: .
    volumes:
       - db_data_sqlite:/var/lib/kutt
       - custom:/kutt/custom
    environment:
      DB_FILENAME: "/var/lib/kutt/data.sqlite"
    ports:
      - 3000:3000
    restart: unless-stopped
volumes:
  db_data_sqlite:
  custom:
```

Reapply:

```bash
docker compose up -d
```

The container will now restart automatically if the system reboots.

---

## 7. Backup Notes

SQLite database and custom files are stored inside `./db` within the container.
To persist data, ensure volumes are declared (example):

```
volumes:
  - ./data:/kutt/db
```

Backup by copying the `data` directory.

---

## 8. Standard Uninstallation

Stop and remove services:

```bash
docker compose down
```

Remove images:

```bash
docker image rm kutt/kutt
```

Remove persisted data if used:

```bash
rm -rf ./data
```

Remove Docker if needed:

```bash
sudo apt purge docker-ce docker-ce-cli containerd.io -y
sudo rm -rf /var/lib/docker /var/lib/containerd
```

