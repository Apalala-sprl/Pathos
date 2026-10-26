# Pathos

**AWS Inspector vulnerabilities of the container images you actually run**: triage,
diff between scans, remediation debt.

[![License: PolyForm Shield 1.0.0](https://img.shields.io/badge/license-PolyForm%20Shield%201.0.0-467082)](LICENSE.md)
[![Image](https://img.shields.io/badge/image-ghcr.io%2Fapalala--sprl%2Fpathos-a3d948)](https://github.com/orgs/Apalala-sprl/packages/container/package/pathos)

Amazon Inspector reports hundreds of CVEs, and most of them concern ECR images that no
longer run anywhere. Pathos keeps the findings that matter and answers three questions:

1. which CVEs affect what is **really running**;
2. which ones are **new** since the previous scan;
3. how many are **still to be triaged** (the debt), the rest being mitigated, false
   positive, not exploitable or accepted.

Pathos is a single, self-contained web application: one Docker image (about 66 MB),
one SQLite database, no external services. It is built in Rust by
[Apalala SRL](https://www.apalala.be).

## Features

| View | What you get |
|---|---|
| **Vulnerabilities** | summary tiles and severity breakdown, filters, sorting, inline triage, a side panel per CVE (description, scores, packages and file paths, affected images, decision history, EUVD/NVD links) |
| **Images** | one row per image, broken down by the artefact that carries the vulnerability (war, jar, binary, OS package) with versions and paths; printable report / PDF |
| **Paths** | flat list of image × path × package × CVE, filterable (`WEB-INF/lib`, no path…) |
| **Diff** | new / gone CVEs and usage changes between the last two comparable scans |
| **Debt** | counter, progress, and a quick-triage queue ranked by urgency (severity, public exploit, number of containers) |
| **Scans** | AWS region and scope, history, manual scan, scheduled scan with catch-up at startup |
| **AWS access** | IAM Identity Center (SSO) login from the UI, fallback STS keys, or the container's own role |
| **Diagnostic** | redacted support report (configuration, scans, recent log) to download, copy or send |

Triage decisions are kept per CVE across scans, with a full history (who, when, why).
CSV and JSON exports are available from the UI and the API.

### What "in use" means

By default a scan imports the findings of **images in use**:

- images running on **ECS or EKS**, according to Inspector's own usage data
  (`ecrImageInUseCount ≥ 1`);
- images behind an **AWS Lambda function**. Inspector does not count Lambda usage, so
  Pathos resolves it itself: every image-packaged function, its unpublished version and
  the versions referenced by its aliases.

You can switch the scope to **all images** that Inspector has scanned. The region and the
scope are chosen in the *Scans* tab. "New" CVEs and the diff always compare two scans of
the same region and scope.

## Installation

### Requirements

- a Linux host with Docker Engine and the Docker Compose v2 plugin;
- outbound access to `ghcr.io` (directly or through the Docker daemon's proxy) and to the
  AWS APIs of your region;
- AWS credentials with the read-only permissions listed in [AWS access](#aws-access).

Install Docker if needed (Debian / Ubuntu shown, other distributions:
<https://docs.docker.com/engine/install/>):

```bash
curl -fsSL https://get.docker.com | sudo sh
```

### Deploy

The image is public: no account and no `docker login` are needed. It also ships its own
deployment files.

```bash
mkdir -p ~/pathos && cd ~/pathos
docker run --rm ghcr.io/apalala-sprl/pathos:1.1.0 compose     > docker-compose.yml
docker run --rm ghcr.io/apalala-sprl/pathos:1.1.0 env-example > .env
```

Generate the instance key and write it into `.env`:

```bash
sed -i "s|^PATHOS_SECRET_KEY=.*|PATHOS_SECRET_KEY=$(docker run --rm ghcr.io/apalala-sprl/pathos:1.1.0 gen-key)|" .env
```

Then edit `.env` and set at least:

- `PATHOS_ADMIN_PASSWORD`: the first administrator's password (12 characters minimum);
- the AWS access you want to use (see below).

Start it:

```bash
docker compose up -d
docker compose ps        # "healthy" after about 15 seconds
```

The UI listens on `http://127.0.0.1:8090`, on the host only. To reach it from your
network, put it behind a TLS reverse proxy and set `PATHOS_PUBLIC_URL=https://…` (secure
cookies, HSTS) and `PATHOS_TRUST_PROXY=true`, or change `PATHOS_BIND` deliberately.

Sign in as `admin`, choose the region and the scope in *Scans*, and run a first scan.

> Keep the instance key (`PATHOS_SECRET_KEY`) with your backups: without it, the SSO
> tokens and fallback keys stored in the database cannot be decrypted.

### Upgrade

Change the version in `PATHOS_IMAGE` (in `.env`), then:

```bash
docker compose pull && docker compose up -d
```

The database schema is migrated automatically. Pin a version (`1.1.0`) rather than
`latest`.

## AWS access

Pathos only reads. Required permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "inspector2:ListFindings",
      "lambda:ListFunctions",
      "lambda:ListAliases",
      "lambda:GetFunction"
    ],
    "Resource": "*"
  }]
}
```

The `lambda:*` permissions are only used to find the images run by Lambda functions
(`GetFunction` is only called on image-packaged functions). Without them, scans still
succeed, with a warning, but without the Lambda images.

For each scan, Pathos picks credentials in this order:

1. **IAM Identity Center (SSO)**: set `PATHOS_SSO_START_URL`, `PATHOS_SSO_REGION`,
   `PATHOS_SSO_ACCOUNT_ID` and `PATHOS_SSO_ROLE_NAME`, then sign in from the *AWS access*
   tab (device code flow: a code and a link to approve in your browser). No AWS CLI and no
   `~/.aws` mount are needed; tokens are encrypted in the database and refreshed
   automatically.
2. **Fallback keys**: temporary STS keys pasted in the *AWS access* tab, validated with
   `sts:GetCallerIdentity` before being stored encrypted.
3. **Environment**: the standard AWS chain (ECS task role, EC2 instance role, EKS web
   identity, `AWS_*` variables).

Behind a corporate proxy, set `HTTPS_PROXY` / `NO_PROXY`. If the proxy intercepts TLS,
mount your CA certificate and set `PATHOS_EXTRA_CA_FILE` (see the comment in
`docker-compose.yml`).

## Configuration

All settings are environment variables in `.env`. Every secret also accepts a `_FILE`
form (Docker / Compose secrets), e.g. `PATHOS_ADMIN_PASSWORD_FILE`.

| Variable | Default | Purpose |
|---|---|---|
| `PATHOS_IMAGE` | — | image to run, pinned to a version |
| `PATHOS_BIND`, `PATHOS_PORT` | `127.0.0.1`, `8090` | where the UI is published on the host |
| `PATHOS_SECRET_KEY` | generated in `/data/secret.key` | instance key encrypting stored secrets |
| `PATHOS_ADMIN_USER`, `PATHOS_ADMIN_PASSWORD` | `admin`, random | first account (ignored once an account exists) |
| `PATHOS_AWS_REGION` | `eu-west-3` | default region, changeable in the *Scans* tab |
| `PATHOS_SCAN_INTERVAL_HOURS` | `24` | scheduled scans (`0` disables them) |
| `PATHOS_SSO_*` | — | IAM Identity Center settings (see above) |
| `PATHOS_PUBLIC_URL` | — | public `https://` URL: enables secure cookies and HSTS |
| `PATHOS_TRUST_PROXY` | `false` | trust `X-Forwarded-For` (behind a reverse proxy only) |
| `PATHOS_SESSION_HOURS` | `12` | session lifetime |
| `PATHOS_API_TOKEN` | — | bearer token for `/api/v1/*` (32 characters minimum) |
| `PATHOS_EXTRA_CA_FILE` | — | extra PEM CA bundle (TLS-intercepting proxy) |
| `HTTPS_PROXY`, `NO_PROXY` | — | outbound proxy |
| `PATHOS_LOG`, `PATHOS_LOG_FORMAT` | `info`, text | log level; `json` for a log collector |
| `PATHOS_SUPPORT_EMAIL`, `PATHOS_SUPPORT_URL` | — | support contact offered on the *Diagnostic* page |

## Security

- **Authentication is mandatory**: local accounts (Argon2id), server-side sessions (only a
  SHA-256 of the 256-bit token is stored), `HttpOnly; SameSite=Strict` cookies, `Secure`
  and `__Host-` prefix over HTTPS. Login throttling: 10 failures per 15 minutes per IP.
- **CSRF**: per-session token required on every state-changing request, plus an origin check.
- **Secrets at rest** (SSO tokens, fallback keys) are encrypted with XChaCha20-Poly1305 and
  never returned by the UI or the API. A wrong instance key is detected at startup.
- **Headers**: strict CSP with no inline script or style, `frame-ancestors 'none'`,
  `nosniff`, `no-referrer`, COOP/CORP, HSTS over HTTPS; authenticated pages are not cached.
- **CSV exports** are protected against formula injection.
- **Container**: distroless image without a shell, non-root user (65532), read-only root
  filesystem, all capabilities dropped, `no-new-privileges`, bounded resources.
- **AWS access** is read-only.

## Operations

```bash
docker compose logs -f --tail=100 pathos
docker compose exec pathos pathos user list
printf '%s' 'a new long password' | docker compose exec -T pathos pathos user set-password admin
docker compose exec -T pathos pathos user delete old-account
```

**Backup.** Triage decisions are the valuable data. Back up the `pathos-data` volume
**and** the instance key:

```bash
docker compose stop pathos
docker run --rm -v pathos-data:/data -v "$PWD":/backup alpine tar czf /backup/pathos-$(date +%F).tgz -C /data .
docker compose start pathos
```

**API.** `/api/v1/export/triage.csv` and `/api/v1/export/triage.json` return all triage
decisions, with a session or `Authorization: Bearer <PATHOS_API_TOKEN>`. Scans can be
started with `POST /api/v1/scans`.

## Support and diagnostics

The *Diagnostic* page builds a report for support: version, configuration, recent scans
and errors, credential status and the last 400 log lines. Secrets, key ids, tokens, AWS
account ids and e-mail addresses are masked. Nothing is sent automatically: download or
copy the report, then attach it to an e-mail or a ticket.

If the UI is unreachable:

```bash
docker compose exec -T pathos pathos diagnostic > pathos-diagnostic.txt   # without the log
docker compose logs --tail=500 pathos > pathos-logs.txt
```

For more detail, set `PATHOS_LOG=debug` and restart.

## Image

`ghcr.io/apalala-sprl/pathos` is published for each release with the tags `X.Y.Z`,
`X.Y`, `X` and `latest`. The OCI labels `org.opencontainers.image.version` and
`org.opencontainers.image.revision` identify the release and the source commit:

```bash
docker buildx imagetools inspect ghcr.io/apalala-sprl/pathos:1.1.0
```

## License

Pathos is licensed under the [PolyForm Shield License 1.0.0](LICENSE.md).
Copyright Apalala SRL.

In short, and without replacing the license text:

- you **may** use Pathos for any purpose, including inside your company and in
  production, and modify it for your own use;
- you **may not** use it to provide a product or service that **competes** with Apalala's
  products, for example by selling or hosting Pathos, or a derivative of it, as an
  offering of your own;
- if you pass the software on, you must pass on the license terms and the
  `Required Notice` line.

The [license text](LICENSE.md) is the only reference. For any other use, such as
commercial redistribution or integration in your own offering, contact
[Apalala SRL](https://www.apalala.be).
