---
title: 'Tutorials | Coolify'
---

## Deploying DMS With Coolify

[Coolify][coolify] deploys this repository with the **Docker Compose** build pack, which makes the [`compose.yaml`][gh-compose] of this repository the single source of truth for the published ports, the persistent volumes and the `NET_ADMIN` capability Fail2Ban needs. Do not duplicate those settings in the Coolify panel.

### Application Settings

| Setting | Value |
| --- | --- |
| Build Pack | `Docker Compose` |
| Branch | `main` |
| Compose File Location | `/compose.yaml` |
| Base Directory | `/` |
| Port Mappings | Leave empty - `compose.yaml` publishes the ports |
| Expose With Traefik | Disabled - DMS is not an HTTP service |

!!! note "The compose file builds the local `Dockerfile`"

    The `build` block builds the image from this repository on the Coolify host, which requires a few GB of free disk space (ClamAV, Amavis and Rspamd packages) and several minutes on the first deployment.

### Environment Variables

Set these in the Coolify panel (they override `mailserver.env`):

| Variable | Example | Description |
| --- | --- | --- |
| `MAIL_HOSTNAME` | `mail.example.com` | Container hostname; must be the FQDN your MX record points to. |
| `ENABLE_FAIL2BAN` | `1` | Enables Fail2Ban. `compose.yaml` already grants the `NET_ADMIN` capability it requires. |
| `SSL_TYPE` | `letsencrypt` | TLS provisioning mode (or `manual` with mounted certificates), see the [SSL/TLS documentation][docs-ssl]. |

Every other variable from `mailserver.env` can be set the same way, but only variables passed through in `compose.yaml` (`environment:`) take effect from the panel while `env_file: mailserver.env` remains in use.

### DNS And Firewall

- `A`/`AAAA` record for `MAIL_HOSTNAME`, plus the `MX` record pointing at it.
- Matching `PTR` (reverse DNS) record - most receivers reject mail without it.
- `SPF`, `DKIM` and `DMARC` records (see the [DKIM documentation][docs-dkim]).
- Open `25`, `143`, `465`, `587` and `993` in the host firewall **and** in the provider's security group. Many cloud providers block port `25` by default.

### Persistence

Mail data, state, logs and the DMS config volume are named volumes (`dms-mail-data`, `dms-mail-state`, `dms-mail-logs`, `dms-config`), so they survive redeployments. Bind mounts into the deployment directory would not, because Coolify clones a fresh artifact directory on every deployment.

Because the config directory is a volume and not a folder in the repository, copy override files into the running container instead of editing them locally:

```bash
docker cp ./postfix-main.cf mailserver:/tmp/docker-mailserver/
docker restart mailserver
```

Back up a volume with:

```bash
docker run --rm -v dms-mail-data:/data:ro -v "${PWD}":/backup alpine \
  tar czf /backup/dms-mail-data.tgz -C /data .
```

### First Steps After Deployment

```bash
# Create the first mailbox (non-interactive). `-c` lets `setup.sh` locate the
# `dms-config` volume through the container, instead of the default local path.
./setup.sh -c mailserver email add user@example.com "your-password"

# Equivalent without the helper script
docker exec mailserver setup email add user@example.com "your-password"

# Check the container health (uses `dms-healthcheck`, 90 s start period)
docker inspect --format '{{.State.Health.Status}}' mailserver
```

[coolify]: https://coolify.io
[docs-dkim]: https://docker-mailserver.github.io/docker-mailserver/latest/config/best-practices/dkim_dmarc_spf/
[docs-ssl]: https://docker-mailserver.github.io/docker-mailserver/latest/config/security/ssl/
[gh-compose]: https://github.com/docker-mailserver/docker-mailserver/blob/main/compose.yaml
