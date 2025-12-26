Consent-based device and event audit logging platform # didactic-chainsaw
# didactic-chainsaw

[![Release](https://img.shields.io/github/v/release/sashameche23/didactic-chainsaw?label=release)](https://github.com/sashameche23/didactic-chainsaw/releases)
[![Build Status](https://img.shields.io/github/actions/workflow/status/sashameche23/didactic-chainsaw/ci.yml?branch=main)](https://github.com/sashameche23/didactic-chainsaw/actions)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](./LICENSE)
[![Issues](https://img.shields.io/github/issues/sashameche23/didactic-chainsaw)](https://github.com/sashameche23/didactic-chainsaw/issues)

Consent-first device & application audit logging platform — secure, tamper-resistant, and privacy-preserving.

Table of contents
- [Overview](#overview)
- [Key features](#key-features)
- [Getting started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Quickstart (Docker Compose)](#quickstart-docker-compose)
  - [Local development](#local-development)
- [Architecture](#architecture)
- [Core concepts](#core-concepts)
- [API](#api)
  - [Authentication](#authentication)
  - [Endpoints & examples](#endpoints--examples)
- [Deployment](#deployment)
  - [Docker](#docker)
  - [Kubernetes / Helm](#kubernetes--helm)
- [Security & privacy](#security--privacy)
- [Operational guidance](#operational-guidance)
  - [Backup & retention](#backup--retention)
  - [Monitoring & alerting](#monitoring--alerting)
- [Contributing](#contributing)
- [License](#license)
- [Legal & ethical use](#legal--ethical-use)
- [Contact](#contact)

Overview
--------
didactic-chainsaw is a consent-based device and event audit logging system. It collects, stores, and serves tamper-resistant audit logs from managed devices and applications while enforcing explicit consent and privacy constraints.

Key features
------------
- Secure event ingestion (mutual TLS / JWT)
- Tamper-resistant append-only logs (WORM-like append, digital signatures)
- Device registration & authentication
- Admin dashboard and search
- Fine-grained consent model per-device / per-user
- Audit trails suitable for compliance (configurable retention, immutability controls)

Getting started
---------------

Prerequisites
- Docker >= 20.x and docker-compose OR Kubernetes cluster (v1.20+)
- PostgreSQL >= 12 (or compatible)
- Redis (optional, for caching/queues)
- TLS certificates for production (Let's Encrypt / internal CA)
- Optional: HSM or KMS for signing keys

Quickstart (Docker Compose)
1. Clone the repo
   ```
   git clone https://github.com/sashameche23/didactic-chainsaw.git
   cd didactic-chainsaw
   ```

2. Create a `.env` file (example values):
   ```
   APP_ENV=development
   DATABASE_URL=postgres://chainsaw:password@db:5432/chainsaw
   REDIS_URL=redis://redis:6379/0
   SIGNING_KEY_PATH=/secrets/signing_key.pem
   JWT_SECRET=change_this_long_random_string
   ```

3. Start with docker-compose:
   ```yaml
   # example docker-compose.yml snippet
   version: "3.8"
   services:
     db:
       image: postgres:14
       environment:
         POSTGRES_DB: chainsaw
         POSTGRES_USER: chainsaw
         POSTGRES_PASSWORD: password
     redis:
       image: redis:6
     chainsaw:
       image: sashameche23/didactic-chainsaw:latest
       env_file: .env
       ports:
         - "8080:8080"
       depends_on:
         - db
         - redis
   ```

   Then:
   ```
   docker-compose up --build
   ```

4. Apply database migrations (if not automatic):
   ```
   docker exec -it <chainsaw_container> chainsaw migrate
   ```

Local development
- Contribute code by forking and creating feature branches.
- Run unit tests: `make test` or `./scripts/test.sh`
- Run linters and formatters: `make lint` / `prettier` / `gofmt` / `black` (depending on project language)

Architecture
------------
High-level components:
- Ingest API: accepts signed, authenticated events from devices/apps.
- Device Management: register devices, manage credentials, enforce consent.
- Audit Store: append-only storage (primary DB with signed log entries and optional WORM-backed object store).
- Search & Query Engine: indexed access for admins (time-range, device, event-type).
- Admin UI: dashboard for consent, device inventory, and audit search.
- Signer Service: signs events and rotates signing keys (HSM/KMS recommended).

Core concepts
-------------
- Device: an enrolled endpoint with credentials and consent scope.
- Event: an action or telemetry point that should be logged (JSON payload).
- Consent Record: explicit grant from a principal allowing particular event categories.
- Audit Entry: stored event with metadata, signature, and ingestion proof.
- Retention Policy: configurable rules controlling how long logs are kept and when they are irrevocably archived.

API
---
Base URL: https://api.example.com (replace with your deployment)

Authentication
- Device-to-server: mTLS or JWT client tokens (recommended: short-lived JWTs issued from a device-auth service).
- Admin: OAuth2 / OpenID Connect with RBAC.

Common headers
- Authorization: Bearer <token>
- Content-Type: application/json
- X-Device-ID: <device-id>
- X-Request-ID: <uuid> (recommended for tracing)

Endpoints & examples

1) Device registration
- POST /v1/devices
- Body:
  {
    "device_id": "device-42",
    "device_type": "linux-agent",
    "metadata": { "owner": "team-a", "location": "datacenter-1" }
  }
- Response: 201 Created with device credentials and enrollment instructions.

2) Ingest event
- POST /v1/events
- Example cURL:
  ```
  curl -X POST "https://api.example.com/v1/events" \
    -H "Authorization: Bearer <DEVICE_JWT>" \
    -H "Content-Type: application/json" \
    -d '{
      "device_id": "device-42",
      "event_type": "user_login",
      "timestamp": "2025-12-26T10:45:00Z",
      "payload": {
        "username": "alice",
        "method": "ssh",
        "success": true
      }
    }'
  ```
- Response: 202 Accepted
  {
    "event_id": "evt_01F...",
    "status": "queued",
    "signed": true
  }

3) Query audit entries (admin)
- GET /v1/audit?device_id=device-42&from=2025-01-01T00:00:00Z&to=2025-12-31T23:59:59Z&page=1&per_page=100
- Response: paginated list, each entry contains a server signature and chain-of-trust metadata.

4) Consent management
- POST /v1/consent
  {
    "subject_id": "user-123",
    "device_id": "device-42",
    "allowed_event_types": ["user_login", "file_access"],
    "granted_at": "2025-12-26T10:00:00Z",
    "expires_at": "2026-12-26T10:00:00Z"
  }

Refer to the OpenAPI / Swagger spec in the repo (docs/openapi.yaml) for full details.

Deployment
----------

Docker
- Build:
  ```
  docker build -t sashameche23/didactic-chainsaw:latest .
  ```
- Run with environment variables and secrets mounted (signing key, TLS certs).
- Use a process supervisor and log aggregator (e.g., systemd + journald -> ELK/Vector).

Kubernetes / Helm
- Provide secrets for DB, JWT signing key, TLS certs (use sealed-secrets or your cluster KMS).
- Deploy with a Helm chart (example path: helm/charts/didactic-chainsaw).
- Recommended:
  - Use StatefulSet for DB components
  - Use Deployment with PodDisruptionBudgets for API and workers
  - Horizontal Pod Autoscaler for API under load
  - NetworkPolicy to restrict traffic to only necessary subnets

Security & privacy
------------------
This project is explicitly consent-first. Before deploying, ensure:
- Explicit, auditable consent flows are implemented for every device and principal.
- All transport is TLS 1.2+ (prefer 1.3) and certificates are validated.
- Device authentication uses short-lived credentials or mTLS.
- Audit logs are append-only, signed, and stored in an immutable-backed store if regulatory immutability is required.
- Use KMS/HSM for signing keys; rotate keys periodically and publish key rotation metadata.
- Encrypt sensitive data at rest (column-level encryption for PII) and in transit.
- Redact or minimize PII in logs where it is not necessary for audit purposes.
- Rate-limit ingestion endpoints and apply DDoS protections.
- Regularly run security scans (SAST/DAST) and dependency checks.

Operational guidance
--------------------

Backup & retention
- Backups: nightly DB dumps + continuous WAL shipping to a secure backup store.
- Retention: configurable per organization; support for archival and legal hold.
- Test restores quarterly.

Monitoring & alerting
- Instrument metrics (Prometheus / OpenTelemetry) for:
  - ingestion latency
  - queue/backlog size
  - signature failure rates
  - storage utilization
- Alert on:
  - high ingestion error rate
  - signing key expiry
  - failed backups

Contributing
------------
We welcome contributions.

- Please read [CONTRIBUTING.md](./CONTRIBUTING.md) for the full process.
- High-level:
  - Fork the repo
  - Create a feature branch: `git checkout -b feat/describe-thing`
  - Add tests and update docs
  - Open a pull request describing the change, linking any related issues
- Code style and tests are enforced in CI.

License
-------
This project is provided under the MIT License. See the [LICENSE](./LICENSE) file for details.

Legal & ethical use
-------------------
This software must only be used on devices and systems that you own, operate, or where you have explicit authorization. All logging and tracking require explicit, legally valid consent where applicable (e.g., employees, customers). The authors make no warranty and are not responsible for misuse.

When deploying in regions with privacy laws (GDPR, CCPA, etc.), ensure:
- you have a lawful basis for processing,
- consent is recorded and revocable,
- data subject rights (access, deletion, portability) can be honored.

Contact
-------
Project maintainer: Sasha Meche (sashameche23)  
Repository: https://github.com/sashameche23/didactic-chainsaw

If you need enterprise support or have questions about compliance and architecture, open an issue or contact via GitHub.

Acknowledgements
----------------
This project was designed as a privacy-respecting audit/logging foundation. Contributions and feedback are welcome.
https://github.com/sashameche23/didactic-chainsaw.git
