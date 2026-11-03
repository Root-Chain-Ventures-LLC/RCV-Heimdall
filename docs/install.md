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

**Connecting Seshat.** Under Settings, Integrations, enter the Seshat address and an API token.
With Seshat 0.5 or newer, ask a Seshat administrator to register Heimdall as an **integration client**
(Settings, Integrations in Seshat) with the capabilities `tracker.read`, `tracker.bulk`, `tracker.poll`
and `events.subscribe` (and `edge.relay.tcp`, read intent only, and `edge.snmp.read` if Heimdall will
reach a remote site through a Seshat Edge; the relay is documented separately), scoped to your
cutover sites, and to issue a token for it. That token cannot change anything in Seshat. A legacy
**`tracker:read`** token still works, without event subscriptions (Heimdall then uses the manual route
under Live events) and, on a Seshat older than 0.5, with one lookup per MAC instead of a batch. An
`operator` token also works but can change Seshat settings that Heimdall never touches, so avoid it.
Seshat allows one poll-now per device every 60 seconds for an API token; a pre-check that hits the
limit waits and retries once, then reads the table Seshat already has.

The Test button reports whether the token can read the Device Tracker, what the token can do (batch
lookups, poll now, event subscription; a missing one is a warning with a sentence, not a failure)
and whether Seshat serves the data formats Heimdall is built for. After the token works, **Event
subscription** (Settings, Integrations, Seshat) has a **Subscribe to events** button: Heimdall creates
its own subscription in Seshat, and after any downtime catches up from Seshat's event log. Seshat must
be able to reach Heimdall's `BASE_URL`; it needs `https`, or an administrator must allow `http` for the
client. No Seshat route or webhook has to be built by hand.

Quick checks from the Docker host:

```sh
curl -fsS http://127.0.0.1:8080/api/health
```

answers `{"status":"ok","version":"<version>","commit":"...","built_at":"..."}`. The same version
is printed by `docker compose run --rm --no-deps heimdall --version`.

## Starting a cutover

A cutover (an outage event) is started with a name and a date: **Cutovers, New cutover**. The time zone is
your browser's; the window times are optional. Nothing else is needed to begin, and everything else is
added as the information arrives:

- **Site.** A cutover with no site shows "Site: not set" and a **Set site** button. Importing the plan, running
  the pre-check and going live need the site (they read that site's switches), and the page says so. A site,
  once set, stays.
- **Notes.** The cutover log has a note box: Enter posts, Shift+Enter starts a new line, and your name and the
  time are saved with the note.
- **Contacts and links.** The crew lead, the customer contact, the contractor, the NOC on duty; the ticket, the
  map, the change record. Links are `http` or `https` only. Operators and administrators edit; viewers read.
- **Meetings.** One pre-cutover and one post-cutover meeting, each with a time, a place or join link, attendees,
  an agenda, notes and decisions, and a status (scheduled, held, cancelled). The suggested time is the working
  day before the window (after, for the post meeting) at 10:00. **Calendar file (.ics)** downloads the meeting;
  open it in your calendar or attach it to a mail. Every change to the meeting raises the file's sequence number,
  so a calendar treats a new download as an update. Heimdall sends no invitations by itself; if an administrator
  has connected Microsoft 365 (Settings, Integrations), a meeting also has **Send invitation through Microsoft
  365**, off by default, and an optional group to invite. Saving then sends, updates or cancels a calendar
  invitation, and the meeting shows its state, the Teams link and **Retry now**. People who got the invitation do
  not need the .ics. See `docs/integrations/microsoft-365.md`.
- The next meeting shows on the cutover page and in the cutovers list. The closing report, with its Excel and PDF
  exports, lists both meetings with their notes and decisions, the contacts and the links.

## Reaching remote sites through a Seshat Edge

Some switches and the ACS sit at sites this server has no route to. If a Seshat Edge (Seshat 0.5.0 or newer) is installed
there, Heimdall can read them through it. The edge dials out only, so the site accepts no inbound connection. Everything
Heimdall sends through an edge is **read only**: SNMP reads (get and walk), SSH show commands and ACS GET requests. It
is off until you set a target's transport to "via Seshat Edge"; every target is read directly until then.

### What to set up in Seshat

1. **An integration client for Heimdall** (Seshat, Settings, Integrations). It holds these capabilities: `tracker.read`,
   `tracker.bulk`, `tracker.poll`, `edge.relay.tcp` and `edge.snmp.read`. Its scope names the cutover sites **and the
   edges** Heimdall may use (a client with no edges in its scope reaches none). Issue one token. **Heimdall uses one
   token for all of it**: the Seshat token already entered under Settings, Integrations. Seshat's integration model gives a client
   one set of capabilities and does not recommend a second token for the relay, so Heimdall has no separate relay token
   field. A legacy `tracker:read` token does not carry the two edge capabilities.
2. **On each edge** (Seshat, Edges, the edge's drawer):
   - **Relay**: mode `read` (Heimdall declares read intent only; `read+write` is not needed) and an **allow-list** of
     `host:port` entries for what Heimdall may reach: the SSH port of each switch (`192.0.2.11:22`, or a network such as
     `192.0.2.0/24:22`) and, for an ACS, its port (`192.0.2.30:7557`). An empty list means the relay is off. A host
     *name* needs an entry pinned to a network (`name@192.0.2.0/24:22`); an address needs none.
   - **SNMP reads**: switched on, with the **networks** the reads may reach in the SNMP allow-list (`192.0.2.0/24`). Heimdall
     seals a one-time SNMP credential to the edge's key for every request, and Seshat only uses a sealed credential
     inside that allow-list (or the edge operator's ceiling). If the switch's SNMP port is not 161, the entry must name
     it (`192.0.2.0/24:1161`). The switch's **management address in Heimdall must be an IP address**: the edge reads by
     address, not by name.
   - Optional but recommended: the edge-local ceilings in the edge's environment (`SESHAT_EDGE_RELAY_CEILING`,
     `SESHAT_EDGE_RELAY_MAX_MODE=read`, `SESHAT_EDGE_SNMP_CEILING`), so even a compromised Seshat server cannot widen
     what the edge reaches.
3. **Read the edge's seal key fingerprint** from its status page (or the Edges page) once, so you can pin it in Heimdall.

### What to set in Heimdall

- **Switches**: Settings, Switches, edit the switch. Next to the management address, **How Heimdall reaches this switch**:
  *Via a Seshat Edge*, and the **edge id** as Seshat shows it. Seshat gives a scoped integration token no way to list
  edges, so the id is typed; **Test edge** asks the edge itself (it proves the id, the scope, that the edge is connected
  and that the token holds `edge.snmp.read`) and shows the fingerprint of the edge's seal key. Compare it with the edge's
  own page and press **Pin this key**: Heimdall then refuses to seal a credential to any other key, which matters
  because the key is handed out by the Seshat server. The transport applies to the SNMP collector, the pre-check's
  MAC-table reader and the SSH fallback (SSH uses the switch's SSH profile as before; its host key is pinned inside the
  stream, on the switch).
- **ACS**: Settings, Integrations, ACS, the same field. If the address is `http://` (not `https://`), the form asks you
  to tick **I understand the relay path can read this**; the API refuses to save without it, and the relay refuses to
  open the stream. Prefer `https://`.
- Nothing else changes: SNMP profiles, SSH profiles and the polling interval are the ones you already have.

### While a cutover is live

When a cutover goes live, Heimdall opens an **SNMP read window** on each edge its switches use (Seshat, `POST
/api/edges/{id}/snmp/windows`): one read per switch per *minimum interval*, which is the polling interval (at least
10 s, Seshat's floor), until the planned end plus 30 minutes (at most 8 hours; with no planned end, 2 hours, renewed
while the cutover stays live). Outside a window Seshat allows one read a minute per switch. The window is not renewed
after the cutover closes and lapses on its own. Each poll reads its OIDs in as few edge requests as the limit of 64 OIDs
allows (the poll learns what it asked for last time and asks for all of it at once), and waits for its turn in the window.
A switch with many watched ports therefore costs more than one request per poll; roughly six watched ports fit in one
request.

When something is wrong on the way, the reading goes **stale or unknown with the reason**, and Heimdall never marks a port
down or a row failed because of it:

| What you see | What it means |
|---|---|
| "The Seshat Edge ... is offline" | The edge has no connection to Seshat (`edge_offline`). Readings resume when it is back. |
| "The relay is switched off" / "does not allow this target" | The edge's relay or SNMP allow-list does not cover the target (`relay_disabled`, `target_not_allowed`; the message names the rule). |
| "SNMP reads are switched off" | The edge's SNMP reads are off (`snmp_disabled`). |
| "cannot use the edge relay" / "scope does not include" | The Seshat token lacks the capability or the edge is outside its scope (`capability_missing`, `out_of_scope`). |
| "is rate limiting the reads" | Over the window rate (`limit_exceeded`, with Retry-After). |
| "The switch did not answer" via the edge | The device itself did not answer the edge (`snmp_timeout`); a v2c switch that does not know the community is silent too. |
| "will not send a credential" | The edge's seal key is not the pinned one (`key_pin_mismatch`). Check the edge, then unpin and pin again. |

### What is logged

Every relayed TCP session (SSH, ACS) writes one line to Heimdall's audit log when it ends (edge, target, protocol, bytes
in and out, duration, outcome), and one when it is refused. SNMP reads are **not** logged per poll: a line is written when
their state changes (the first read works, a failure begins or changes, recovery) and a summary every 15 minutes while
they run and when the cutover ends; the same lines go to the event log of the live cutovers at that site. Window opens,
renewals and failures are logged. Payloads, SNMP values and credentials are never logged. Seshat audits the same traffic
on its side (`edge.relay`, `edge.snmp.read`).

### Exposure, and the threat model

- **Read only is Heimdall's rule, Seshat enforces what it can.** The SNMP path is enforced at the edge: the request
  schema has get, getnext, getbulk and walk and nothing else. The TCP relay is a byte stream: Seshat records the
  declared intent (`read`) and the edge checks it against its mode, but neither can see inside. What keeps SSH and
  HTTP read only is Heimdall: SSH runs only commands on the fixed per-vendor allow-list of read-only show commands, the ACS client
  only sends GET. A stolen Seshat token can open only the targets in the edge's allow-list, within its limits, and every
  use is audited by Seshat.
- **SNMP v2c.** The community is sealed to the edge's key, so it never crosses the relay path in the clear and the
  Seshat server cannot read it. From the edge to the switch it is v2c on the site's own network, as it would be with
  a direct read. Prefer SNMPv3 where the switch supports it.
- **SSH and https** run end to end through the stream: Seshat and the edge see ciphertext, and the credentials never
  leave Heimdall. **Plain http** (an ACS without TLS) is plain text on the Seshat server and on the way to it, which
  is why it needs the per-target acknowledgement.
- **The seal key comes through the Seshat server**; a compromised server could offer its own. Pin the fingerprint.
- **A compromised Seshat server** can allow any target the edge can reach unless the edge has a local ceiling; set one.
- Heimdall keeps no write path to any device: there is no SNMP set, no configuration command and no ACS write in
  any transport.

## Single sign-on (optional)

Heimdall can sign people in with any OpenID Connect provider and give each person a role from their groups there. Local
password sign-in keeps working; the first administrator is always local.

1. At the provider, register Heimdall as a web application (authorization code flow) with the redirect URI
   `<BASE_URL>/api/auth/oidc/callback` (Settings > Sign-in shows it, built from `BASE_URL`). Note the client ID and create a client secret.
2. Make the provider put the person's groups into the ID token (a `groups` claim; the claim name is configurable).
3. In Heimdall, Settings > Sign-in: enter the issuer URL exactly as the provider publishes it, the client ID and secret, and under
   "Groups to roles" the groups for admin, operator and viewer (one per line). Press Test to read the provider's discovery
   document, save, then switch Single sign-on on. Someone in none of the listed groups cannot sign in; the highest role wins.
4. Keep "Allow password sign-in" on while you try it. Turned off, only administrators can use a password, at
   `<BASE_URL>/sign-in?local=1`; that is the way in if the provider is down or misconfigured.

`HEIMDALL_SSO_SESSION_TTL` (for example `8h`, from `5m` to `24h`, default 8 h) sets how long a single sign-on session lasts.

**Microsoft Entra ID.** Issuer `https://login.microsoftonline.com/<tenant-id>/v2.0`. Under Token configuration add a groups claim
for security groups, as Group ID, and limit it to groups assigned to the application, which avoids group overage (a token with
too many groups carries a link instead of the list, and Heimdall does not follow it). Put the group object IDs in the three lists.
Entra sends no `email_verified` claim: to link an existing local account by email, tick "Trust the email address when the provider
sends no email_verified claim" under More options. New accounts need nothing.

An existing account that has a password, or is an administrator, is not linked by email alone: open Settings > Users, choose
"Allow SSO sign-in for this account" on it, and its owner is linked at their next verified single sign-on. Until then they see "An
administrator must allow single sign-on for this account first." (Accounts with no password that are not administrators link
without this step.) An allowed local administrator keeps the administrator role whatever their groups say.

People who sign in this way are created on their first sign-in (switch it off to allow only people who already have an account),
have no Heimdall password, and are marked SSO under Settings > Users. Their role follows their groups at every sign-in.

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
you took in step 1 and start the old bundle. Starting the old binary against the upgraded database
(a binary-only rollback) works only for releases whose migrations are all expand-only; a release that
ships a contract step (a migration whose first line is `-- contract:`) says so in its release notes.

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
