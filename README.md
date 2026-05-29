# Form.io Deployment Examples

This repository provides one Docker Compose stack that runs the [Form.io](https://form.io) Enterprise Platform locally with no configuration choices to make. It bundles Caddy as the reverse proxy and SeaweedFS as the S3 backend, alongside the Form.io API server, PDF server, and MongoDB.

> ⚠️ **These are examples, not production deployments.** Default secrets, single-node databases, and unencrypted HTTP are used throughout for simplicity. Review [Production Considerations](#production-considerations) before running anything customer-facing.

## Prerequisites

- **Docker Engine 24.x or newer** with **Docker Compose v2** (the `docker compose` subcommand, not the legacy `docker-compose` binary). [Docker Desktop](https://docs.docker.com/desktop/) ships both.
- **A valid Form.io Enterprise license key.** The containers will refuse to start without one. Contact [sales@form.io](mailto:sales@form.io) to obtain a key.
- Roughly **2 GB of free RAM** and a few GB of disk for images and volumes.

> **Apple Silicon / ARM note:** The Form.io images are built for `linux/amd64`. Docker Desktop on ARM Macs runs them under emulation automatically, so first boot is slower than on x86.

## Quickstart

```bash
# 1. Clone and enter the quickstart directory
git clone https://github.com/formio/evaluation-quickstart.git
cd evaluation-quickstart

# 2. Create your .env from the template
cp .env.example .env

# 3. Open .env and set your license key (required):
#      LICENSE_KEY=<your-key-here>
#
#    Change the default secrets before first run:
#      ADMIN_PASS, JWT_SECRET, DB_SECRET, FORMIO_S3_SECRET

# 4. Start the stack
docker compose up -d

# 5. Watch for the ready banner
docker compose logs -f ready-message
```

When you see `🚀 Form.io is now fully up and running!`, open **http://localhost:3000** and sign in with the `ADMIN_EMAIL` and `ADMIN_PASS` set in `.env`.

To stop the stack:

```bash
docker compose down       # stop containers, preserve data volumes
docker compose down -v    # stop containers AND delete all data
```

## Environment Variables

The source of truth is [docker/quickstart/.env.example](docker/quickstart/.env.example). The most important variables:

| Variable | Default | Required | Description |
|---|---|---|---|
| `LICENSE_KEY` | *(empty)* | **Yes** | Form.io Enterprise license. Containers fail fast if unset. |
| `DB_SECRET` | `123secret123` | Yes | Encrypts sensitive fields in MongoDB. **Do not change after data is written.** |
| `JWT_SECRET` | `123secret123` | Yes | Signs JWTs. Changing this invalidates all active sessions. |
| `ADMIN_EMAIL` | `admin@example.com` | Yes | Root admin account email, created on first boot. |
| `ADMIN_PASS` | `CHANGEME` | Yes | Root admin account password. |
| `ADMIN_KEY` | `thisIsMyXAdminKey` | Yes | Static API key for admin-level programmatic access. |
| `MONGO_DATABASE` | `quickstart` | Yes | MongoDB database name. |
| `MONGO_USER` / `MONGO_PASSWORD` | `admin` / `admin` | Yes | MongoDB root credentials. |
| `FORMIO_S3_BUCKET` | `files` | Yes | Bucket name for uploads. |
| `FORMIO_S3_REGION` | `us-east-1` | Yes | Region label (local S3 stores usually ignore this). |
| `FORMIO_S3_KEY` / `FORMIO_S3_SECRET` | `admin@example.com` / `CHANGEME` | Yes | S3 access credentials, shared between SeaweedFS and the PDF server. |

Image tags use fallback syntax in the compose file (`${API_SERVER_VERSION:-9.7}`), so you can override them in `.env` without editing `docker-compose.yml`.

## Architecture

Incoming traffic hits Caddy on the host port (default `3000`). Caddy routes `/pdf/*` to the PDF server and everything else to the API server.

```
        Browser
          │
          ▼
   ┌──────────────┐
   │    Caddy     │  :3000 on host → :80 inside container
   └──┬────────┬──┘
      │        │
  /*  │        │  /pdf/*
      ▼        ▼
┌──────────┐ ┌──────────┐
│   API    │ │   PDF    │
│  Server  │ │  Server  │
│  :5000   │ │  :5000   │
└────┬─────┘ └────┬─────┘
     │            │
     ▼            ▼
┌──────────┐ ┌────────────┐
│ MongoDB  │ │ SeaweedFS  │
│  :27017  │ │   :9000    │
└──────────┘ └────────────┘
```

| Container | Image | Role |
|---|---|---|
| `formio-proxy` | `caddy:2-alpine` | Path-based routing |
| `formio-api` | `formio/formio-enterprise:9.7` | Core API and Developer Portal |
| `formio-pdf` | `formio/pdf-server:5.13` | PDF generation, form rendering to PDF |
| `formio-db` | `mongo:8.2` | Document store for all platform data |
| `formio-s3` | `chrislusf/seaweedfs` | S3-compatible file and PDF storage |
| `ready-message` | `alpine:latest` | One-shot container that prints a success banner once the API and PDF healthchecks pass |

Services use healthcheck-based dependency ordering. Mongo starts first. The API waits for a healthy Mongo. The PDF server waits for both the API and SeaweedFS. The ready-message container waits for the API and PDF to be healthy.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Container exits with `Contact sales for a valid license key.` | `LICENSE_KEY` is blank in `.env` | Paste your key and restart. |
| `formio-api` constantly restarts | Conflicting configuration changes | Use `docker compose down -v` to clear existing volumes and restart. |
| Healthcheck never passes; stack hangs | Mongo has not initialized (cold pull, low RAM) | Run `docker compose logs formio-db` and give it 60 seconds on first run. |
| 502 / Bad Gateway from Caddy | API still starting up | The API healthcheck has a 30-second `start_period`. Wait and retry. |
| Encrypted fields unreadable after restart | `DB_SECRET` was changed | Restore the original value. There is no recovery without the original secret. |
| Sessions lost on every restart | `JWT_SECRET` changed between runs | Set a stable `JWT_SECRET` in `.env`. |
| Slow startup on Apple Silicon | `linux/amd64` emulation | Expected behavior. First boot can take 1 to 2 minutes. |

## Production Considerations

This stack is designed for local evaluation. Before running Form.io in any real environment:

- **Replace every default secret.** Generate strong values with `openssl rand -hex 32`. This applies to `DB_SECRET`, `JWT_SECRET`, `ADMIN_PASS`, `ADMIN_KEY`, `FORMIO_S3_SECRET`, and `MONGO_PASSWORD`.
- **Use a managed MongoDB** such as Atlas, DocumentDB, or CosmosDB. The bundled container has no replication, no automated backups, and weak credentials.
- **Use managed object storage** such as AWS S3, Azure Blob, or GCS. The single-node SeaweedFS container is not durable.
- **Pin all image tags.** Avoid `:latest` in any environment. Use full versions such as `:9.7.8`.
- **Set resource limits** on each container to prevent runaway memory consumption.
- **Back up MongoDB and S3 data** regularly, and test your restores.
- **Consult the [Form.io Self-Hosted Deployment Guide](https://help.form.io/deployments/deployment-guide)** for the complete production checklist, including multi-environment licensing, Kubernetes, and Terraform patterns.

## Support

> A more configurable example with swappable reverse proxy and S3 backends is also in this repo but will be moved to it's own repo at a future date.

| Channel | When to use |
|---|---|
| 🐛 [GitHub Issues](https://github.com/formio/evaluation-quickstart/issues) | Bugs or suggestions for these examples |
| 📚 [help.form.io](https://help.form.io) | Official platform documentation |
| 📧 [sales@form.io](mailto:sales@form.io) | License keys, pricing |
| 🛟 [support@form.io](mailto:support@form.io) | Paid customer support |

## License

This repository is licensed under the [MIT License](LICENSE).

The Form.io Enterprise Platform itself is licensed separately. A valid Enterprise license key is required to run the containers referenced in these examples.
