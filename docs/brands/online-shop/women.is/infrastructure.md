# women.is Infrastructure Overview

**Last updated: February 13, 2026**

---

## SERVER

- Provider: Contabo VPS
- Hostname: vmi2978954
- IP: 185.230.138.60
- OS: Ubuntu 24

---

## ARCHITECTURE

```
Internet → Global Caddy (ports 80/443) → Docker network "web" → Store containers
```

One global Caddy container handles TLS and routing for ALL domains.
Each store has its own `db` + `wp` container pair.
All containers share the `web` Docker network.

---

## GLOBAL CADDY

- Location: `/opt/caddy/`
- Container name: `caddy`
- Config: `/opt/caddy/Caddyfile`
- Compose: `/opt/caddy/docker-compose.yml`

### docker-compose.yml

```yaml
services:
  caddy:
    image: caddy:2
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"
    networks:
      - web
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
      - women_women_wp:/var/www/html/women.is:ro
      # ADD new store volumes here when creating new stores
networks:
  web:
    external: true
volumes:
  caddy_data:
  caddy_config:
  women_women_wp:
    external: true
  # ADD new store volume declarations here
```

### Caddyfile

```
n8n.kolvetni.is {
    reverse_proxy n8n-n8n-1:5678
}
baserow.kolvetni.is {
    reverse_proxy baserow:80
}
mcp.kolvetni.is {
    reverse_proxy mcp:3000
}
www.women.is {
    redir https://women.is{uri} permanent
}
women.is {
    root * /var/www/html/women.is
    php_fastcgi women-wp-1:9000 {
        root /var/www/html
    }
    file_server
}
# NEW STORES GET APPENDED HERE BY new-store.sh
```

### Useful Caddy commands

```bash
# Reload config (no downtime)
docker exec caddy caddy reload --config /etc/caddy/Caddyfile

# Watch logs
docker logs -f caddy 2>&1 | grep -E 'error|certificate|obtain'

# Restart Caddy (clears TLS backoff)
cd /opt/caddy && docker compose down && docker compose up -d
```

---

## STORE TEMPLATE

- Location: `/opt/stores/women-template/`
- Used as base for ALL new stores via `cp -r`

### docker-compose.yml (template)

```yaml
name: ${STORE_NAME}
services:
  db:
    image: mariadb:10.11
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - web
  wp:
    image: wordpress:php8.3-fpm
    restart: unless-stopped
    depends_on:
      - db
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_NAME: ${DB_NAME}
      WORDPRESS_DB_USER: ${DB_USER}
      WORDPRESS_DB_PASSWORD: ${DB_PASSWORD}
    volumes:
      - wp_data:/var/www/html
    networks:
      - web
volumes:
  db_data:
  wp_data:
networks:
  web:
    external: true
```

---

## ACTIVE STORES

### women.is

| Property       | Value                        |
|----------------|------------------------------|
| Domain         | https://women.is             |
| Admin          | https://women.is/wp-admin    |
| Admin user     | admin                        |
| Admin email    | 8242766@gmail.com            |
| Store dir      | /opt/stores/women-template   |
| WP container   | women-wp-1                   |
| DB container   | women-db-1                   |
| WP volume      | women_women_wp               |
| DB volume      | women_women_db               |
| DB name        | wp_women                     |
| DB user        | wp_women_user                |

**Plugins installed:**
- WooCommerce 10.5.1
- CJ Dropshipping (connected via Default OAuth)
- Stripe: pending

**Notes:**
- women.is uses `women-template` as its directory (not a separate folder)
- DB user was created manually (volume existed from old ghost containers)
- WP-CLI installed at `/usr/local/bin/wp` inside women-wp-1

### Other services (kolvetni.is)

| Domain                 | Container    | Port |
|------------------------|--------------|------|
| n8n.kolvetni.is        | n8n-n8n-1    | 5678 |
| baserow.kolvetni.is    | baserow      | 80   |
| mcp.kolvetni.is        | mcp          | 3000 |

---

## CREATING A NEW STORE (automated)

### One-command setup

```bash
/opt/stores/new-store.sh <store_name> <domain> "<site_title>" <admin_email> <db_password>
```

**Example:**
```bash
/opt/stores/new-store.sh airfryer airfryer.is "Airfryer Shop" 8242766@gmail.com MySecurePass123
```

### What new-store.sh does automatically

1. Copies women-template → /opt/stores/<store_name>
2. Writes .env with DB credentials
3. Starts db + wp containers
4. Waits 20s for DB to initialize
5. Installs WP-CLI
6. Adds domain to /opt/caddy/Caddyfile
7. Mounts wp volume into global Caddy (updates docker-compose.yml)
8. Recreates Caddy container
9. Runs WordPress core install
10. Installs + activates WooCommerce
11. Creates WooCommerce pages, sets permalinks

### After new-store.sh completes

- DNS: point <domain> A record → 185.230.138.60
- TLS: Caddy auto-issues Let's Encrypt cert once DNS propagates
- CJ Dropshipping: connect manually via https://app.cjdropshipping.com → Add Store
- Stripe: add manually via WooCommerce → Payments

### Helper script

```
/opt/stores/update-caddy-compose.py  — called by new-store.sh to update caddy compose
```

---

## BACKUPS

### What gets backed up

- `db.sql` — full mysqldump of the WordPress database
- `wp-content.tar.gz` — all uploads, themes, plugins

### Schedule

- **When:** Every night at 02:00
- **Location:** `/opt/backups/<store_name>/<date>/`
- **Retention:** 7 days (older folders deleted automatically)

### Cron job

```bash
# View current cron
crontab -l

# Entry:
0 2 * * * /opt/backups/backup.sh >> /var/log/backup.log 2>&1
```

### Backup script location

```
/opt/backups/backup.sh
```

### Manual backup run

```bash
/opt/backups/backup.sh
```

### Check backup logs

```bash
cat /var/log/backup.log
```

### Check backup files

```bash
ls -lh /opt/backups/women/
ls -lh /opt/backups/women/2026-02-13/
```

### Restore database (example)

```bash
docker exec -i women-db-1 mysql -uwp_women_user -p<password> wp_women < /opt/backups/women/2026-02-13/db.sql
```

### Restore wp-content (example)

```bash
docker run --rm \
  --volumes-from women-wp-1 \
  -v /opt/backups/women/2026-02-13:/backup \
  alpine tar xzf /backup/wp-content.tar.gz -C /
```

---

## USEFUL COMMANDS

### General

```bash
# All running containers
docker ps

# All volumes
docker volume ls | grep women
```

### Per-store

```bash
# WP-CLI (replace 'women' with store name)
docker exec women-wp-1 wp --allow-root <command>

# DB shell
docker exec -it women-db-1 mysql -uwp_women_user -p<password> wp_women

# Restart store
cd /opt/stores/women-template && docker compose restart

# View WP logs
docker logs -f women-wp-1
```

### WP-CLI examples

```bash
# Install plugin
docker exec women-wp-1 wp plugin install <slug> --activate --allow-root

# List plugins
docker exec women-wp-1 wp plugin list --allow-root

# Update all plugins
docker exec women-wp-1 wp plugin update --all --allow-root

# Create admin user
docker exec women-wp-1 wp user create bob bob@example.com --role=administrator --allow-root
```

---

## FILE MAP

```
/opt/
├── caddy/
│   ├── Caddyfile              # Global reverse proxy + TLS config
│   └── docker-compose.yml     # Global Caddy container (add new store volumes here)
├── stores/
│   ├── women-template/        # women.is store (also base template)
│   │   ├── docker-compose.yml
│   │   └── .env
│   ├── new-store.sh           # One-command new store generator
│   └── update-caddy-compose.py # Helper: adds volume to caddy compose
└── backups/
    ├── backup.sh              # Nightly backup script
    └── women/
        └── 2026-02-13/
            ├── db.sql
            └── wp-content.tar.gz
```

---

## ADDING STRIPE (when ready)

1. Go to https://women.is/wp-admin → WooCommerce → Payments
2. Enable Stripe
3. Add API keys from https://dashboard.stripe.com
4. Enable only EUR currency
5. Test with Stripe test mode before going live

---

## CJ DROPSHIPPING NOTES

- Connected to women.is via Default OAuth
- Filter products by: **Shipping From → DE / EU warehouses**
- Main EU warehouses: Germany (DE), France (FR), Poland (PL)
- Orders fulfil automatically once CJ is connected and products imported
