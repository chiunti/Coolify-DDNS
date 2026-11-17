# Namecheap Dynamic DNS (DDNS) for Coolify

A lightweight Alpine-based container that keeps a Namecheap DNS record pointing at
your current public IPv4 address — a self-hosted alternative to DynDNS-style
services, deployable as a single Docker Compose stack.

## How it works

1. On startup, it resolves the IP currently published for your FQDN via `1.1.1.1`
   (so it doesn't push a redundant update if the record is already correct).
2. Every `CHECK_INTERVAL` seconds it fetches your public IP (with fallback across
   `api.ipify.org`, `ifconfig.me`, `icanhazip.com`) and validates it is a real IPv4.
3. Only when the IP **changes** does it call the Namecheap Dynamic DNS endpoint
   (`dynamicdns.park-your-domain.com`) and verifies `<ErrCount>0</ErrCount>` in the
   XML response. Failed updates are retried on the next cycle.
4. Each loop iteration writes a heartbeat file used by the Docker healthcheck;
   `autoheal` restarts the container if it ever reports `unhealthy`
   (e.g. a stuck loop — something `restart: always` alone cannot detect).

## 1. Namecheap configuration (required)

> **Important:** Namecheap Dynamic DNS only works when the domain uses
> **Namecheap's own nameservers** (BasicDNS / PremiumDNS / Web Hosting DNS).
> If the domain points to custom nameservers (e.g. Cloudflare), this will not
> work — use that provider's API instead.

1. Log in to Namecheap → **Domain List** → **Manage** your domain.
2. Make sure the **Nameservers** section is set to *Namecheap BasicDNS* or
   *Namecheap Web Hosting DNS*.
3. Go to the **Advanced DNS** tab.
4. Scroll to **Dynamic DNS** and toggle it **ON** (Status: Enabled).
5. Copy the **Dynamic DNS Password** shown there — this is your `DDNS_PASSWORD`.
   It is **not** your account password; it's a dedicated per-domain password that
   works for all dynamic records on that domain.
6. In **Host Records** (same Advanced DNS tab), add a record of type
   **A + Dynamic DNS**:
   - **Host:** `@` for the root domain, or a subdomain name (e.g. `home`,
     `nas`). This value is your `DDNS_HOST`.
   - **Value:** any placeholder IPv4 (e.g. `127.0.0.1`) — the script overwrites
     it on first sync.
   - **TTL:** *Automatic* is fine.
7. Save and allow a few minutes for propagation.

## 2. Environment variables

| Variable         | Required | Default | Description                                              |
|------------------|----------|---------|----------------------------------------------------------|
| `DDNS_DOMAIN`    | Yes      | —       | Your domain, e.g. `example.com`                          |
| `DDNS_PASSWORD`  | Yes      | —       | Dynamic DNS Password from Advanced DNS                   |
| `DDNS_HOST`      | No       | `@`     | Host of the A + Dynamic DNS record (`@` = root domain)   |
| `CHECK_INTERVAL` | No       | `120`   | Seconds between public IP checks                         |

Example `.env`:

```dotenv
DDNS_DOMAIN=example.com
DDNS_PASSWORD=your-dynamic-dns-password
DDNS_HOST=@
CHECK_INTERVAL=120
```

> Never commit your `.env` — the DDNS password is a secret.

## 3. Deployment

### Option A — Coolify

1. In Coolify: **+ Add Resource** → **Docker Compose (empty)**.
2. Paste the contents of `docker-compose.yaml`.
3. In the service's **Environment Variables** section, define `DDNS_DOMAIN`,
   `DDNS_PASSWORD` and (optionally) `DDNS_HOST` / `CHECK_INTERVAL`.
   Mark `DDNS_PASSWORD` as a secret if available.
4. **Deploy**. Open the container logs — a healthy start looks like:

   ```
   [DDNS] Namecheap Dynamic DNS service started for: example.com
   [DDNS] Initial IP resolved from DNS (1.1.1.1): 203.0.113.10
   [DDNS] Public IP unchanged (203.0.113.10). No request sent to Namecheap.
   ```

5. Coolify surfaces the healthcheck status; if the service turns `unhealthy`,
   autoheal restarts it automatically.

### Option B — Plain Docker Compose

```bash
cp .env.example .env   # then fill in your values
docker compose up -d
docker logs -f namecheap-ddns
```

## Subdomains

The updater only manages the single record pointed to by `DDNS_HOST` — other
records are untouched, so subdomains don't follow the apex IP by themselves.

The easy fix is CNAMEs pointing at the apex. In **Advanced DNS → Host Records**
add a **CNAME Record** per subdomain (or a `*` wildcard to cover them all):

| Type | Host | Value | Covers |
|---|---|---|---|
| CNAME Record | `home` | `example.com` | `home.example.com` |
| CNAME Record | `*` | `example.com` | every undefined subdomain |

When `@` is updated, every CNAME'd subdomain resolves to the new IP
automatically — no extra containers needed.

Alternative (only if a subdomain must point to a *different* IP than the apex):
create an **A + Dynamic DNS** record for it and run another instance of this
stack with that `DDNS_HOST` — this script manages one host per container.

## Autoheal (auto-restart on unhealthy)

Docker's `restart: always` only handles process exits — a container stuck in an
unhealthy state stays running. The bundled `willfarrell/autoheal` service watches
the Docker socket and restarts any container labeled `autoheal=true` once it
reports `unhealthy` (after a 120s grace period).

- If you already run autoheal on the host, you can delete the bundled service —
  the `autoheal=true` label on `namecheap-ddns` is all it needs.
- **Security note:** mounting `/var/run/docker.sock` grants root-equivalent
  control of the Docker host. This is inherent to autoheal; deploy it only on a
  host you trust.

## Troubleshooting

| Log message | Meaning |
|---|---|
| `[FATAL] DDNS_DOMAIN and DDNS_PASSWORD are required` | Missing env vars — set them and redeploy |
| `[ALERT] No internet connection or all IP providers failed` | Outage or all 3 IP providers unreachable — retries automatically |
| `[WARN] Namecheap request failed (curl error N)` | Couldn't reach `dynamicdns.park-your-domain.com` — check the container's outbound connectivity; retries automatically |
| `[ERROR] Namecheap rejected the update` | Namecheap answered with `ErrCount > 0`; the extracted `Err1`/`ResponseString` lines give the exact cause. Common ones: `Passwords do not match` → wrong `DDNS_PASSWORD`; `Domain name not found` → `DDNS_DOMAIN` isn't the apex domain or isn't on Namecheap DNS; `No Records updated. A record not Found;` → no *A + Dynamic DNS* record exists for `DDNS_HOST` |
| Container `unhealthy` | The loop hasn't iterated within `2 × CHECK_INTERVAL + 120s`; inspect with `docker inspect --format '{{json .State.Health}}' namecheap-ddns` |

## Limitations

- IPv4 only — Namecheap Dynamic DNS supports A records, not AAAA.
- The initial sync trusts `1.1.1.1`'s cached answer; with very low TTLs this is
  accurate within seconds, and a stale read at worst triggers one harmless update.
