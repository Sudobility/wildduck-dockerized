# CLAUDE.md — wildduck-dockerized (0xmail fork)

> **Git policy — never auto-commit or auto-push.** Leave work in the working tree.
> Only run `git commit`/`git push`/`gh pr create` when the user asks in that turn.

## What this is

Docker Compose deployment of the **WildDuck** mail server stack, extended with a
**blockchain Mail Box Indexer** for wallet-based email addresses (`0xmail.box`).

- **Fork**: `github.com/Sudobility/wildduck-dockerized`, derived from upstream
  `nodemailer/wildduck-dockerized`. Ships custom `johnqh/*` images + an indexer + Postgres.
- **Not a code package.** There is **no `package.json`, no test suite, no build step,
  no TypeScript** in this repo. It is Bash (`setup.sh`, `upgrade.sh`, `versions.sh`,
  `monitor-containers.sh`, `setup-scripts/*.sh`) + YAML/TOML config + Dockerfiles.
  Ignore any older doc (including files under `.claude/`) that mentions `npm test`, Jest,
  `npm run deploy`, `tsconfig`, or a `tests/` directory — none exist here.
- The repo root `Dockerfile` just packages `setup.sh` into an Alpine image; it is not the
  mail server. Custom service images are built from `docker-images/{haraka,rspamd,zone-mta}`.
- License: EUPL-1.2.

## Stack (default `docker-compose.yml`, 10 services)

Network `wildduck_network`, subnet `172.20.0.0/16`.

| Service | Image | Notes |
|---|---|---|
| wildduck | `johnqh/wildduck:latest` | IMAP(143)/POP3(110)/API(8080). Static IP 172.20.0.10 |
| zonemta | `ghcr.io/zone-eu/zonemta-wildduck:1.32.20` | Outbound SMTP 587, DKIM sign |
| haraka | `johnqh/haraka:latest` | Inbound SMTP, port 25 published directly |
| rspamd | `nodemailer/rspamd` | Spam filter, runs as UID 11333 |
| mail_box_indexer | `johnqh/mail_box_indexer:latest` | Blockchain indexer, port 42069. Static IP 172.20.0.20 |
| mongo | `mongo` | Shared by wildduck/zonemta/haraka |
| postgres | `postgres:15` | Indexer DB only (Ponder) |
| redis | `redis:alpine` | Sessions, queues, rate limiting |
| traefik | `traefik:2.11` | Reverse proxy + TLS. **2.11 LTS on purpose** (3.x had Docker API issues) |
| static-files | `nginx:alpine` | Serves `robots.txt` for API domain |

`docker-compose-alt-ports.yml` is a **different, lighter upstream-style stack** (adds
webmail, no indexer/postgres/static, Traefik **3.3.4**, HTTP/HTTPS on **8090/8443**) — not
just a port remap. Don't conflate the two files.

### Ports
Published: 25 (haraka direct), 80/443 (traefik; 80→443 redirect), 465/993/995 (traefik TCP
passthrough → zonemta 587 / wildduck 143 / wildduck 110). Localhost-only: 8080 (API),
5432 (postgres). Internal-only: mongo 27017, redis 6379, rspamd 11333/11334.

### Traefik routing
Host+path on `API_HOSTNAME`: `/api/*`→wildduck:8080 (strip `/api`), `/idx/*`→indexer:42069
(strip `/idx`), `/robots.txt`→static. TCP `imaps`/`pop3s`/`smtps` match `HostSNI(HOSTNAME)`.
`HOSTNAME` (mail host) and `API_HOSTNAME` (api host) are placeholders replaced by `setup.sh`.

## Deploy & operate

```bash
sudo ./setup.sh              # interactive wizard (Linux/macOS); generates config-generated/
./upgrade.sh                 # pull latest images, keep config+secrets, restart
./versions.sh                # versions of running containers
./monitor-containers.sh full # diagnostics

# Compose commands run from INSIDE config-generated/ (not repo root):
cd config-generated
sudo docker compose up -d
sudo docker compose logs -f <service>
sudo docker compose restart <service>
sudo docker compose down
```

## setup.sh flow (critical, non-obvious)

1. Sources `setup-scripts/{deps_setup,kill_ports}.sh`; wipes any existing `config-generated/`.
2. **Doppler is central**: prompts for a Doppler service token, downloads WildDuck/indexer
   secrets via the Doppler API into `.env`. Token cached in `.doppler-token` (chmod 600).
   Doppler values override existing `.env` on merge.
3. Prompts: mail domain (required), mail hostname, API hostname, indexer private key/wallet
   (if not from Doppler), CORS toggle, self-signed-certs toggle (`y` dev / `N` prod).
4. Copies `default-config/` → `config-generated/config/`, `docker-compose.yml` →
   `config-generated/`; `sed`-replaces `HOSTNAME`/`API_HOSTNAME` in the generated compose.
5. Auto-detects remote Mongo/Postgres URLs and **comments out the local DB services** in the
   generated compose.
6. Self-signed mode: generates rootCA + host cert under `config-generated/certs/`, patches
   `dynamic_conf/dynamic.yml`. Prod mode: uncomments the Let's Encrypt resolver lines.
7. Prod only: writes `update_certs.sh` + a weekly cron that extracts LE certs from Traefik's
   `acme.json` for Haraka (Haraka manages its own certs separately).
8. `docker compose up -d` from `config-generated/`.
9. `setup-scripts/dns_setup.sh`: generates 1024-bit DKIM key (selector `<mon><year>`, e.g.
   `oct2025`), registers it with WildDuck via API, prints MX/SPF/DKIM/DMARC records.

Note: `setup-scripts/user_setup.sh` exists but is **not** invoked by `setup.sh`. There is no
interactive admin-user prompt — the root account is a `rootUsername` wallet address in
`api.toml`; create users via the WildDuck API.

## Config layout

- Edit **templates** in `default-config/<service>/` (used on the next setup run).
- **Active** config after setup is the generated copy: `config-generated/config/<service>/`.
  WildDuck configs are TOML (`default.toml`, `dbs.toml`, `api.toml`, `imap.toml`,
  `dkim.toml`, …). Editing `config-generated/config/...` + restart is the live-change path.
- `default-config/mail_box_indexer/.env` is copied to repo-root `.env` as a starting point.

## Shared-secret / inter-service gotchas

- **SRS, DKIM, HMAC secrets must match** across WildDuck, ZoneMTA, and Haraka, or mail
  signing/delivery breaks. DKIM secret must be identical in `zone-mta/plugins/wildduck.toml`
  and `wildduck/dkim.toml`.
- WildDuck, ZoneMTA, and Haraka must share the **same MongoDB + same DB name**.
- Redis is shared; services separate concerns by DB index.
- WildDuck↔Indexer auth is **IP-restricted** (wildduck 172.20.0.10, indexer 172.20.0.20) —
  changing static IPs/subnet breaks authentication.
- Haraka TLS is **independent of Traefik** — certs live in `config/haraka/` and are refreshed
  by the `update_certs.sh` cron, not by Traefik.
- WildDuck env `APPCONF_api_indexerBaseUrl` / `APPCONF_*` override TOML config via env vars.

## Environment (`.env.example`, 60+ vars; indexer-focused)

Required: `DATABASE_URL` (Postgres for indexer), `INDEXER_PRIVATE_KEY`,
`INDEXER_WALLET_ADDRESS`, at least one RPC key (`ALCHEMY_API_KEY`, `ANKR_API_KEY`,
Chainstack per-chain endpoints). Chains: Ethereum, Polygon, Optimism, Base, Arbitrum
(+ testnets via `ENABLE_TESTNETS`). Tuning via `PONDER_*`. Optional integrations: Sumsub
(KYC, `KYC_ENABLED`), Helius/Solana, JWT (OAuth), RevenueCat. Defaults are in the compose
`environment:` block with `${VAR:-default}` fallbacks.

## Reference docs
`README.md` (user-facing deploy guide), `ENDPOINTS.md` (full API reference),
`DNS.md`, `DATABASE-DEBUG.md` (+ `debug-db-connection.js`), `mongodb/` & `postgres/`
(init/tuning/backup).
