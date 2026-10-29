# RCV Heimdall

**Heimdall** is a fiber cutover planning and live-watch tool for active-Ethernet ISPs. It replaces the
per-cutover spreadsheet: you import the cutover plan, Heimdall matches every row to a switch port, MAC
address, service address and billing account, records the state of the switches before the window, and then
watches each port through the cutover. The NOC can report, per customer, when the link dropped, when it came
back, what the light levels and error counters look like, and whether the customer's device is back online.

Every row keeps its **before** and **after** state side by side; the before face is never overwritten when the
after face is confirmed.

Heimdall only reads from the network. It sends no SNMP SET, runs no configuration commands and never shuts or
re-enables a port: it observes and records, and people act.

## What it does

- **Plan and import.** Sites, cutovers, phases and rows. Spreadsheet import with column mapping and a preview;
  the messy parts of a real sheet (MACs in mixed formats, free-text addresses, text in numeric columns, merged
  headers) are normalised on the way in, and everything Heimdall writes out is canonical.
- **Pre-check.** Before the window, every row is checked against the switches' MAC tables, from a Seshat
  Device Tracker or read directly over SNMP. Findings are listed per row and kept in the report.
- **Live watch.** During the window a read-only SNMP collector tracks link state, optical power and error
  counters per watched port; link events and streaming telemetry from Seshat refine them. Thresholds turn each
  row green, amber or red, and a row is suggested for verification once it has been up, in range and quiet for the
  time you set. The live board updates without a page reload.
- **Customer device state.** A GenieACS integration shows whether the customer's ONT or modem is back online.
- **Report.** An exceptions-first closing report with the readings kept for each verified or failed row, exports
  to `report.xlsx` and `report.pdf`, and optional forwarding of events to a syslog server (RFC 5424 over UDP, TCP or
  TLS) and to Seshat.
- **Vendors.** Optics readers over SNMP for Dell (OS9), Arista, Juniper, TP-Link and Ciena; a read-only SSH fallback
  (allow-listed `show` commands, pinned host key) where a vendor offers nothing else.
- **Roles.** `admin` (settings, integrations, credentials), `operator` (plan, pre-check, watch, report) and
  `viewer` (reports only).

## License

**RCV Community License 1.1**: free for personal and home-lab use and for any organization's own internal
operations. A paid commercial license is required only to offer the software to third parties as a hosted,
managed, or SaaS service. See [`LICENSE`](LICENSE). Commercial inquiries: **legal@rootchainventures.com**.

Heimdall is built from a private source repository; the source is not published. See
[`CONTRIBUTING.md`](CONTRIBUTING.md).

## What you download

Each release carries the install artifacts and nothing else (the release notes are empty on purpose):

| Artifact | |
|---|---|
| `heimdall-<version>-install.tar.gz` | `compose.yaml` pinned to the release's image by digest, `.env.example` and the install guide |
| `heimdall-<version>-linux-amd64.tar.gz` | the static binary (web UI and database migrations embedded; it still needs a PostgreSQL) and the licence |
| `SHA256SUMS`, `SHA256SUMS.sig`, `cosign.pub` | checksums, their signature and the public key |
| `heimdall-<version>-source.spdx.json`, `heimdall-<version>-image.spdx.json` | software bills of materials for the build inputs and for the image |

The container image (linux/amd64) is pulled without logging in:

- `ghcr.io/root-chain-ventures-llc/heimdall`

Only the newest stable release is kept here; older releases and tags are removed when a new one is published.

## Requirements

- A Linux host with Docker Engine and the Compose plugin (`docker compose version` prints a v2 version).
- A DNS name and a TLS certificate. Heimdall serves plain HTTP and expects a **reverse proxy to terminate TLS**
  in front of it ([`docs/install.md`](docs/install.md) has Caddy and nginx examples).
- Network reach from the Docker host to the switches (SNMP, and SSH if you use the fallback) and to any Seshat,
  NetBox, Nautobot or GenieACS server you connect.

## Install

Download `heimdall-<version>-install.tar.gz` and `SHA256SUMS` from the [latest release](https://github.com/Root-Chain-Ventures-LLC/RCV-Heimdall/releases/latest),
verify them (below), then:

```sh
tar -xzf heimdall-<version>-install.tar.gz
cd heimdall-<version>-install
cp .env.example .env
chmod 600 .env
# put two generated secrets into .env:
#   HEIMDALL_SECRET_KEY   openssl rand -base64 48
#   POSTGRES_PASSWORD     openssl rand -hex 16
docker compose up -d
docker compose logs heimdall        # prints the one-time setup token on first start
```

The web port is published on the loopback interface only, because Heimdall speaks plain HTTP: put a TLS proxy
in front of it before exposing it. Open `BASE_URL` (default `http://localhost:8080`); the first visit asks for the
**setup token**, your organization's name and the first administrator.

**Back up `HEIMDALL_SECRET_KEY` separately from the database.** It seals session cookies and encrypts the stored
credentials of your integrations; without it they cannot be read.

The full guide, with TLS proxy examples, upgrade, backup and restore, is [`docs/install.md`](docs/install.md).

## Verify what you downloaded

Every release is signed with a key pair; the public key is [`cosign.pub`](cosign.pub) in this repository. Take it
from here (or from a release you verified earlier), not from the download being checked.

```sh
cosign verify-blob --key cosign.pub --signature SHA256SUMS.sig --insecure-ignore-tlog=true SHA256SUMS
sha256sum -c --ignore-missing SHA256SUMS
cosign verify --key cosign.pub --insecure-ignore-tlog=true ghcr.io/root-chain-ventures-llc/heimdall:<version>
```

`--insecure-ignore-tlog=true` is expected: signatures are made offline with a key pair and are not uploaded to a
public log. The image also carries an SPDX SBOM and a build-provenance attestation, which
`cosign verify-attestation --type spdxjson` and `--type slsaprovenance1` check against the same key. A release
without `SHA256SUMS.sig` is not a Heimdall release: do not install it.

## Upgrade

1. Take a backup of the database and keep `.env`.
2. Download the new install bundle, check it, unpack it next to the old one, copy your `.env` over and compare it
   with the new `.env.example`.
3. `docker compose pull && docker compose up -d`. Migrations run on start and cannot be reversed; to go back,
   restore the backup and start the old bundle.

## Documentation

[`docs/README.md`](docs/README.md) indexes the operator documentation.

## Status

Heimdall's switch, telemetry and device readers have been verified against simulators built from vendor MIBs and
recorded sample data, and against virtual network devices (SNMP v2c and v3, real link loss and recovery, LLDP
neighbours). Transceiver power (optics) readings have not yet been confirmed on physical hardware. Check each source
against your own equipment before you rely on it in a live window.

## Reporting problems

- **Security issues:** please report them privately, see [`SECURITY.md`](SECURITY.md). Do not open a public issue.
- **Bugs and feature requests:** GitHub Issues on this repository. Use synthetic data (no real device names,
  addresses, MAC addresses, account numbers or credentials).
- **Commercial licensing:** legal@rootchainventures.com.
