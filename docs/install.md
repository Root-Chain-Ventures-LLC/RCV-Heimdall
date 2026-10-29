# Installing Heimdall from the release bundle

This guide installs a released Heimdall on one Docker host: the app and its PostgreSQL database,
two containers, started from the `heimdall-<version>-install.tar.gz` that is attached to the
release. Nothing is built on your machine.

Check what you downloaded before you unpack it. `SHA256SUMS` proves that the files arrived intact; a
release that also carries `SHA256SUMS.sig` proves who built them. Releases published in the public
RCV-Heimdall repository are signed, and that repository's README ("Verify what you downloaded") has the
`cosign` commands and the public key.

## What you need

- A Linux host with Docker Engine and the Compose plugin (`docker compose version` prints a v2
  version).
- A DNS name for Heimdall and a TLS certificate for it (see "TLS in front"), unless you only
  want to reach it from the Docker host itself.
- Network reach from the Docker host to the things Heimdall watches: the switches (SNMP, and SSH
  if you use the read-only fallback), and any Seshat, NetBox or ACS server you connect. Heimdall
  only reads from devices.

## 1. Unpack the bundle

```sh
sha256sum -c --ignore-missing SHA256SUMS     # run next to the downloaded files
tar -xzf heimdall-<version>-install.tar.gz
cd heimdall-<version>-install
```

The folder holds `compose.yaml`, `.env.example` and `docs/`. `compose.yaml` pulls the release's image
by digest, so what runs is exactly what was built.

## 2. Configure `.env`

```sh
cp .env.example .env
chmod 600 .env
```

Edit `.env` and fill in the two secrets, plus the address people will use:

| Variable | What to put | How to make it |
|---|---|---|
| `HEIMDALL_SECRET_KEY` | At least 32 characters of random data. Seals session cookies and encrypts stored integration credentials (a Seshat token, SNMP communities and so on). | `openssl rand -base64 48` |
| `POSTGRES_PASSWORD` | The database password. Letters and digits only. | `openssl rand -hex 16` |
| `BASE_URL` | The address people type into the browser, for example `https://heimdall.example.com`. State-changing requests must carry exactly this origin, and an `https://` address turns on the Secure cookie flag. | |

Keep a copy of `.env` somewhere safe, apart from the database backups. Losing
`HEIMDALL_SECRET_KEY`, or changing it, signs everyone out and makes the stored integration
credentials unreadable; an administrator then has to enter them again under Settings,
Integrations. The other variables in the file (`HEIMDALL_SETUP_TOKEN`, `TRUST_PROXY`,
`HEIMDALL_PUBLISH_ADDR`) are explained in its comments.

## 3. Start it

```sh
docker compose up -d
docker compose ps                      # both services running; heimdall becomes "healthy"
docker compose logs heimdall
```

The app waits for the database, applies its migrations and starts listening. The web port is
published on the loopback interface only (`127.0.0.1:8080`), because Heimdall speaks plain HTTP;
see "TLS in front" to reach it from other machines.

## 4. First-run setup

On a fresh database no administrator exists. Find the one-time setup token in the log:

```sh
docker compose logs heimdall | grep -i setup_token
```

(Or set `HEIMDALL_SETUP_TOKEN` in `.env` before the first start and use that value.) Open
`BASE_URL` in a browser, enter the setup token, the organization name, and the first
administrator's name, email and password. The token is only asked for while no administrator
exists. Administrators then add operators and viewers under Settings, and add sites, switches,
SNMP profiles and integrations.

Quick checks from the Docker host:

```sh
curl -fsS http://127.0.0.1:8080/api/health
```

answers `{"status":"ok","version":"<version>","commit":"...","built_at":"..."}`. The same version
is printed by `docker compose run --rm --no-deps heimdall --version`.

## TLS in front

Put a reverse proxy that terminates TLS in front of Heimdall and keep the app on loopback. Two
settings matter:

1. Set `BASE_URL` to the public https address, and `TRUST_PROXY` to the proxy's address (or `1`
   for one proxy on a private address). Without `TRUST_PROXY` every visitor looks like the proxy,
   so all users share one sign-in rate limit and a few wrong passwords lock everyone out.
2. The live board uses a long-lived event stream (`/api/cutovers/<id>/stream`). The proxy must
   pass it through without buffering and without a short read timeout.

Caddy (runs on the same host, so `TRUST_PROXY=127.0.0.1`):

```
heimdall.example.com {
    reverse_proxy 127.0.0.1:8080 {
        flush_interval -1
    }
}
```

nginx:

```
server {
    listen 443 ssl;
    server_name heimdall.example.com;
    # ssl_certificate and ssl_certificate_key as usual

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_buffering off;
        proxy_read_timeout 1h;
    }
}
```

If the proxy runs on another host, set `HEIMDALL_PUBLISH_ADDR` in `.env` to the one private
address of the Docker host that the proxy reaches (for example `192.0.2.10`), set `TRUST_PROXY`
to the proxy's address, and allow only the proxy to connect to port 8080 in the host firewall.
Never publish the plain HTTP port on an untrusted network.

## Upgrade

1. Read the release notes and `CHANGELOG.md` for the new version. Take a backup first (below).
2. Download the new `heimdall-<new-version>-install.tar.gz`, check it against `SHA256SUMS`, and
   unpack it next to the old folder.
3. Copy your `.env` into the new folder. Compare it with the new `.env.example` and add any new
   variables.
4. From the new folder: `docker compose pull && docker compose up -d`. The Compose project is
   named `heimdall` by the `name:` line in `compose.yaml`, whatever the folder is called, so the
   new containers reuse the existing database volume.
5. Check `curl -fsS http://127.0.0.1:8080/api/health` shows the new version.

Migrations run when the new version starts and are not reversible. To go back, restore the backup
you took in step 1 and start the old bundle.

## Backup and restore

Everything Heimdall stores is in the PostgreSQL database; the only other thing to keep is `.env`.

```sh
# backup (custom format, compressed); run in the folder that holds compose.yaml
docker compose exec -T db pg_dump -U heimdall -d heimdall -Fc > heimdall-$(date +%F).dump

# restore into an empty database (new host, or after "docker compose down -v")
docker compose up -d db
docker compose exec -T db pg_restore -U heimdall -d heimdall --no-owner --clean --if-exists < heimdall-2026-01-01.dump
docker compose up -d
```

Copy the dump off the host, store it with `.env` (the dump holds integration credentials in
encrypted form; they are readable only with the same `HEIMDALL_SECRET_KEY`), and try a restore
now and then.

## Uninstall

```sh
docker compose down            # stops and removes the containers, keeps the database volume
docker compose down -v         # also deletes the database volume: all Heimdall data is gone
docker image rm <the image reference from compose.yaml>
```

Take a backup before `down -v`.
