# Identiqa NMS — Zabbix Docker Stack (host-run license agent)

Docker Compose stack for a branded **Zabbix 7.4** server ("Identiqa NMS"). In
this variant, the Identiqa **license agent runs on the host** (for example from
cron) instead of in a container.

> For the fully containerised version (license agent and license proxy inside
> Compose), see the `zabbix-docker` repository.

## Services (`docker-compose.yaml`)

| Service | Image | Port | Purpose |
|---|---|---|---|
| `mysql-server` | `mysql:8.0` | internal | Zabbix database |
| `zabbix-server` | `zabbix/zabbix-server-mysql:7.4-alpine-latest` | `10051` | Zabbix server |
| `zabbix-web` | `zabbix/zabbix-web-nginx-mysql:7.4-alpine-latest` | internal `8080` | Zabbix frontend with Identiqa branding |
| `zabbix-agent` | `zabbix/zabbix-agent:alpine-7.4-latest` | `10060` → 10050 | Agent monitoring the stack itself |
| `zabbix-snmptraps` | `zabbix/zabbix-snmptraps:latest` | `162/udp` | SNMP trap receiver |

`zabbix-web` only exposes port 8080 inside the Docker network. Put
`license-proxy/` (or another reverse proxy) in front of it to reach the UI.

## Other components in this repo

| Path | Purpose |
|---|---|
| `license-agent/licensev1.mjs` | Checks the license, starts/stops the `zabbix-server` container and reports host count and version to the license API |
| `license-proxy/` | Express proxy (port 3333) that only forwards to the Zabbix UI while the license is valid; otherwise serves `custom.html` |
| `license/license.lic` | License file: key on line 1, instance ID on line 2 |
| `zabbix-custom-ui/` | Branding (logo, favicon, theme, `brand.conf.php`), `zabbix_server.conf`, `snmptrapd.conf` |

## Requirements

- Docker Engine with the Compose plugin
- Node.js 20+ on the host (for the license agent)
- Outbound HTTPS to the license API

## Setup

```bash
git clone https://github.com/identiqalabs/zbxdocker.git
cd zbxdocker

# 1. License: key on line 1, instance ID on line 2
printf '%s\n%s\n' "<LICENSE_KEY>" "<INSTANCE_ID>" > license/license.lic

# 2. Start Zabbix
docker compose up -d

# 3. Install dependencies for the host-side license agent
npm install
```

### Schedule the license agent

Run it every 10 minutes with cron (`crontab -e`). Replace `/path/to/zbxdocker`
with where you cloned the repo:

```cron
*/10 * * * * cd /path/to/zbxdocker/license-agent && /usr/bin/node licensev1.mjs >> /path/to/zbxdocker/licensev1.log 2>&1
```

The agent reads `../license/license.lic` relative to `license-agent/`, and
needs permission to run `docker` commands.

## Configuration

- **Database passwords**: set in `docker-compose.yaml` (`MYSQL_ROOT_PASSWORD`,
  `MYSQL_PASSWORD`). Change them before the first start.
- **Zabbix login**: default is `Admin` / `zabbix`. Change it after first login.
  If you do, update `ZABBIX_USER` / `ZABBIX_PASSWORD` in `license-agent/licensev1.mjs`.
- **Time zone**: `TZ` / `PHP_TZ` (default `Asia/Kolkata`).

## Useful commands

```bash
docker compose ps
docker compose logs -f zabbix-server
tail -f licensev1.log
```
