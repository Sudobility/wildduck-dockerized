# WildDuck Dockerized (0xmail fork) - 🦆 + 🐋 = ❤

A Docker Compose deployment of the [WildDuck](https://wildduck.email) mail server stack,
extended with a **blockchain Mail Box Indexer** for wallet-based email addresses (the
`0xmail.box` system). This is a fork maintained by Sudobility
(`github.com/Sudobility/wildduck-dockerized`); it is derived from the upstream
[nodemailer/wildduck-dockerized](https://github.com/nodemailer/wildduck-dockerized)
but ships custom images (`johnqh/*`), a blockchain indexer, PostgreSQL, and a
Doppler-driven setup wizard.

> No STARTTLS. TLS is SSL/TLS only.

## What's in the stack

The default `docker-compose.yml` runs **10 services** on a dedicated bridge network
(`wildduck_network`, subnet `172.20.0.0/16`):

| Service          | Image                                      | Purpose                                                      |
| ---------------- | ------------------------------------------ | ------------------------------------------------------------ |
| WildDuck         | `johnqh/wildduck:latest`                   | IMAP / POP3 server + REST API (static IP `172.20.0.10`)      |
| ZoneMTA          | `ghcr.io/zone-eu/zonemta-wildduck:1.32.20` | Outbound SMTP: queue, DKIM-sign, deliver                     |
| Haraka           | `johnqh/haraka:latest`                     | Inbound SMTP (port 25), hands mail to Rspamd + WildDuck      |
| Rspamd           | `nodemailer/rspamd`                        | Spam filtering, DKIM/SPF/DMARC checks (runs as UID 11333)    |
| Mail Box Indexer | `johnqh/mail_box_indexer:latest`           | Blockchain indexer for wallet email addresses + OAuth/points (static IP `172.20.0.20`) |
| MongoDB          | `mongo`                                    | Users, mailboxes, messages, queues (shared)                  |
| PostgreSQL       | `postgres:15`                              | Mail Box Indexer database (Ponder framework)                 |
| Redis            | `redis:alpine`                             | Sessions, rate limiting, ZoneMTA queue (shared)              |
| Traefik          | `traefik:2.11`                             | Reverse proxy + TLS termination + Let's Encrypt              |
| Static Files     | `nginx:alpine`                             | Serves `robots.txt` for the API domain                       |

`docker-compose-alt-ports.yml` is a separate, lighter upstream-style stack (WildDuck +
webmail + ZoneMTA + Haraka + Rspamd + MongoDB + Redis + Traefik **3.3.4**, no indexer or
PostgreSQL) that binds HTTP/HTTPS to **8090/8443** instead of 80/443 for running locally
alongside other web servers.

### Ports

| Port         | Service                       | Exposure                          |
| ------------ | ----------------------------- | --------------------------------- |
| 25           | Haraka (inbound SMTP)         | Published directly on the host    |
| 80 / 443     | Traefik (HTTP/HTTPS, API)     | Published; 80 redirects to 443    |
| 465          | SMTPS → ZoneMTA (587)         | Via Traefik (`smtps` entrypoint)  |
| 993          | IMAPS → WildDuck (143)        | Via Traefik (`imaps` entrypoint)  |
| 995          | POP3S → WildDuck (110)        | Via Traefik (`pop3s` entrypoint)  |
| 8080         | WildDuck API                  | Bound to `127.0.0.1` only         |
| 5432         | PostgreSQL                    | Bound to `127.0.0.1` only         |

MongoDB, Redis, and Rspamd are internal to the Docker network and are not published.

### Routing

Traefik routes HTTP(S) by host and path prefix:

- `https://API_HOSTNAME/api/*` → WildDuck API (`:8080`, `/api` prefix stripped)
- `https://API_HOSTNAME/idx/*` → Mail Box Indexer (`:42069`, `/idx` prefix stripped)
- `https://API_HOSTNAME/robots.txt` → static-files
- `imaps`/`pop3s`/`smtps` TCP routers match `HostSNI(HOSTNAME)` and pass through TLS

`HOSTNAME` (mail hostname, e.g. `mail.0xmail.box`) and `API_HOSTNAME` (e.g.
`api.0xmail.box`) are placeholders substituted by `setup.sh`.

## Prerequisites

- Docker and Docker Compose
- A domain name with DNS you control, and a DNS A record pointing at the server
- Ports 25, 80, 443, 465, 993, 995 free (no nginx/Plesk/etc. holding them)
- A **Doppler service token** — `setup.sh` downloads WildDuck/indexer secrets (Mongo URL,
  shared secrets, RPC keys, etc.) from Doppler. This is required for a normal run.
- For the blockchain indexer: an indexer wallet (private key + address) and at least one
  blockchain RPC provider API key (Alchemy, Ankr, Chainstack, …). See `.env.example`.
- For production TLS: ports 80/443 reachable so Traefik can solve the Let's Encrypt
  challenge. For local dev, the wizard can generate self-signed certs instead.

## Deploy with `setup.sh`

`setup.sh` is an interactive wizard (Linux/macOS, run with `sudo`). It generates a
ready-to-run deployment under `config-generated/`.

```bash
git clone https://github.com/Sudobility/wildduck-dockerized.git
cd wildduck-dockerized
sudo ./setup.sh
```

The wizard will:

1. Prompt for your **mail domain** (required), **mail hostname** and **API hostname**
   (optional; default to `mail.<domain>` / `api.<domain>`).
2. Prompt for a **Doppler service token** and download secrets into `.env` (the token is
   cached in `.doppler-token`, `chmod 600`; Doppler values take precedence on merge).
3. Prompt for indexer credentials (private key / wallet address) if not supplied by Doppler.
4. Ask whether to enable CORS for the WildDuck API.
5. Ask whether to generate **self-signed certs** for development (answer `N` for a public
   server using Let's Encrypt; `y` for local dev).
6. Copy `default-config/` → `config-generated/config/`, copy `docker-compose.yml` →
   `config-generated/`, substitute `HOSTNAME`/`API_HOSTNAME`, and auto-detect remote
   MongoDB/PostgreSQL URLs (commenting out the local DB services when a remote URL is used).
7. For production, install an `update_certs.sh` helper plus a weekly cron job that extracts
   Let's Encrypt certs from Traefik's `acme.json` for Haraka (Haraka manages its own certs).
8. `docker compose up -d` the whole stack from `config-generated/`.
9. Generate DKIM keys, register the DKIM key with WildDuck via its API, and print the DNS
   records you need to add.

> Haraka handles TLS certificates separately from Traefik. On a server the installed cron
> keeps Haraka's certs in sync; for a custom setup place certs in `config/haraka/`.

## After setup: DNS records

The wizard prints a `NAMESERVER SETUP` section (MX, SPF, DKIM, DMARC). Add these to your
DNS zone:

```text
<domain>.                         IN MX  5 <hostname>.
<domain>.                         IN TXT "v=spf1 ... ip4:<SERVER_IP> ~all"
<selector>._domainkey.<domain>.   IN TXT "v=DKIM1;k=rsa;p=<PUBLIC_KEY>"
_dmarc.<domain>.                  IN TXT "v=DMARC1; p=reject;"
```

- The DKIM selector is `<mon><year>` (e.g. `oct2025`); keys are 1024-bit RSA.
- Also set a PTR (reverse DNS) record for your server IP at your hosting provider.

## Configuration

Service config templates live in `default-config/` (edited **before** setup). After a run,
the *active* configuration is the generated copy under `config-generated/config/<service>/`:

- `config-generated/config/wildduck/` — WildDuck TOML (`default.toml`, `dbs.toml`,
  `api.toml`, `imap.toml`, `dkim.toml`, …)
- `config-generated/config/haraka/`, `config-generated/config/zone-mta/`,
  `config-generated/config/rspamd/`, `config-generated/config/static/`

To change configuration, edit files under `config-generated/config/` and restart the
affected service:

```bash
cd config-generated
sudo docker compose restart <service>   # e.g. wildduck
# or
sudo docker compose down && sudo docker compose up -d
```

### Thunderbird with self-signed certs

Import the generated CA so the client trusts the mail server: find it at
`config-generated/certs/rootCA.pem`. With Let's Encrypt on a public domain no import is
needed.

## Mail Box Indexer

The indexer enables blockchain-based (wallet) email addresses by indexing smart contracts
across chains (Ethereum, Polygon, Optimism, Base, Arbitrum; optional testnets). It is
configured entirely through environment variables — see the heavily documented
`.env.example` for all 60+ options. Key ones:

```bash
DATABASE_URL=postgresql://ponder:password@postgres:5432/mail_box_indexer  # required
INDEXER_PRIVATE_KEY=0x...       # required (indexer wallet)
INDEXER_WALLET_ADDRESS=0x...    # required
ALCHEMY_API_KEY=...             # at least one RPC provider required
EMAIL_DOMAIN=0xmail.box
ENABLE_TESTNETS=false
```

Optional integrations: Sumsub (KYC, `KYC_ENABLED=true`), Helius/Solana, JWT (OAuth),
RevenueCat (subscriptions). The indexer API is reached at `https://API_HOSTNAME/idx`
externally, `http://mail_box_indexer:42069` from other containers, or `localhost:42069`
if you uncomment its port mapping in the compose file.

## Using the API

- WildDuck REST API: `https://API_HOSTNAME/api` — authenticate with the
  `X-Access-Token: <accessToken>` header. The access token, if configured, is in
  `config-generated/config/wildduck/api.toml` (`accessToken`); the admin/root account is
  identified by `rootUsername` (a wallet address in this fork). See the
  [WildDuck API docs](https://docs.wildduck.email/docs/category/wildduck-api).
- Indexer API: `https://API_HOSTNAME/idx` (REST + GraphQL; wallet-signature / Bearer auth).
- Create a user/mailbox via the API, e.g.:

  ```bash
  curl -XPOST https://api.<domain>/api/users \
    -H 'Content-Type: application/json' \
    -H 'X-Access-Token: <token>' \
    -d '{"username":"alice","password":"pass","address":"alice@<domain>"}'
  ```

A full endpoint reference lives in `ENDPOINTS.md`.

## Operations

```bash
./versions.sh                 # Print versions of all running containers
./upgrade.sh                  # Pull latest images and restart, preserving config + secrets
./monitor-containers.sh full  # Health/diagnostics for all containers
```

Day-to-day Compose commands are run from inside `config-generated/`:

```bash
cd config-generated
sudo docker compose ps
sudo docker compose logs -f <service>
sudo docker compose restart <service>
sudo docker compose down
```

## Further docs

- `DNS.md` — detailed DNS record reference
- `ENDPOINTS.md` — full WildDuck + indexer API reference
- `DATABASE-DEBUG.md` — database connectivity troubleshooting (`debug-db-connection.js`)
- `mongodb/` and `postgres/` — DB init scripts, tuning configs, backup helpers
- Upstream project docs: <https://docs.wildduck.email>
