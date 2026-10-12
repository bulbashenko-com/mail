# mail

[Stalwart](https://stalw.art) mail server of bulbashenko.com. This repo is the Kubernetes base; [bulbashenko-com/infra](https://github.com/bulbashenko-com/infra) deploys it, pinned to one commit (`clusters/hel1/mail.yaml`).

## Shape

| | |
|---|---|
| Image | `stalwartlabs/stalwart` (tag in `deploy/k8s/stalwart.yaml`) |
| Storage | **Postgres** (`config.json`: database `stalwart` on the cluster's CloudNativePG). Settings, accounts, mail, blobs and the search index all live there; the pod is stateless and backed up with Postgres (WAL + daily base backups, encrypted off-site copy). |
| Node | `node.bulbashenko.com/role=mail`: the node whose public IP is mail.bulbashenko.com (MX, SPF, PTR). Outgoing mail leaves from that IP. |
| Ports | host ports `25` (SMTP), `465` (submissions), `993` (IMAPS). Clients use 465/993 with OAuth (Ory) or app passwords. |
| HTTP | `8080` behind Traefik: `autoconfig.` / `autodiscover.` (public, only the auto-configuration paths, via Cloudflare Tunnel) and `mail.int.bulbashenko.com` (admin + web UI, mesh only + Oathkeeper + Stalwart's own OIDC login). |

## Provided by infra

- Namespace `mail`, the node label, and the Secret `stalwart`: `STALWART_DB_PASSWORD`, `CF_DNS_API_TOKEN` (Cloudflare, zone DNS edit, used by Stalwart's DNS automation and DNS-01 ACME).

## Runbook

**What Stalwart manages itself.** DNS records MX, SPF, DKIM (rotated every 90 days), the `mail` TXT and `_smtp._tls`; and its TLS certificates via ACME DNS-01. Keep its Cloudflare token valid; don't edit those records by hand (they are excluded from OpenTofu in infra). DMARC is set by hand.

**Settings.** `config.json` only says where the database is; everything else is in the database and is changed through the web UI (`mail.int`) or `stalwart-cli` against `https://mail.int.bulbashenko.com` with an API key. `stalwart-cli snapshot <types…>` exports settings as an idempotent plan for `stalwart-cli apply` (secrets come back as `****` and must be supplied separately).

**Upgrade.** Change the image tag (PR), then promote the commit in infra. Read the release notes first: Stalwart migrates its schema on start.

**Restore.** Restore the Postgres database (CNPG recovery from the barman object store), then start this Deployment. Moving to another node = move the role=mail label *and* the IP (or update MX/SPF/PTR).

**Migration (2026-10).** From the Coolify container (RocksDB) with `stalwart --export` (server stopped) and `stalwart --import` into Postgres, run once by infra.
