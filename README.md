# Nextcloud AIO: Two rootless Docker engines on one VPS

This project deploys two **independent Nextcloud AIO instances** under separate rootless Docker engines on one Linux VPS. It does **not** deploy both AIO instances under the same Docker engine or Dokploy Compose service. Existing rootful Dokploy, Traefik and regular Nextcloud installations remain running.

## Design

- Linux user `aio-ben`: rootless Docker engine #1, dedicated volumes and AIO containers.
- Linux user `aio-colleague`: rootless Docker engine #2, dedicated volumes and AIO containers.
- Dokploy's EXISTING system Traefik terminates HTTPS and proxies to each AIO Apache host port.
- Each user uses the SAME `compose.yaml` from this repository with a DIFFERENT uncommitted `.env`.
- AIO creates its database, Redis, web app and optional Office/Talk containers within each rootless engine.
- AIO admin UIs are bound to localhost; reach via SSH tunnels.

## Prerequisites and guardrails

- Read [AIO multiple instances](https://github.com/nextcloud/all-in-one/blob/main/multiple-instances.md) and [AIO rootless](https://github.com/nextcloud/all-in-one/blob/main/docker-rootless.md).
- **NEVER disable, stop, uninstall or replace the system Docker daemon**. Dokploy and the existing Nextcloud containers depend on it.
- AIO normally recommends disabling an existing rootful Docker service during rootless installation. **Do not do that here**; when installing each user's rootless engine use `dockerd-rootless-setuptool.sh install --force` as the Docker rootless docs instruct for a coexisting daemon.
- First verify `uidmap`, `newuidmap`, `newgidmap`, subordinate UID/GID allocation, Ubuntu AppArmor requirements and `docker-ce-rootless-extras`. Prefer distro packages rather than piping downloaded scripts into a shell.
- Open a **genuine SSH login as each AIO user** (not only `sudo -iu`, which may lack a user-systemd session and `XDG_RUNTIME_DIR`).
- Enable linger with `sudo loginctl enable-linger aio-ben` and `sudo loginctl enable-linger aio-colleague`, allowing both rootless Docker services to start at boot.
- Each rootless account must use `DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock` and must verify `docker info` displays `rootless` under security options.
- Do not add either user to the host's `docker` group.
- Provide two **different DNS names**, never reuse an existing Nextcloud domain.

## Port assignments

| Function | aio-ben | aio-colleague |
| --- | --- | --- |
| AIO admin UI (SSH tunnel only) | 8080 | 8081 |
| AIO Apache (Traefik backend) | 11000 | 11001 |
| Talk (optional TCP + UDP) | 3478 | 3479 |

Verify all ports are free and block public access to the Apache TCP ports **at the VPS provider/upstream firewall**. The Apache endpoints must remain reachable internally by Traefik. Distinct Talk ports need firewall rules if the Talk backend is enabled.

## How to install each account's rootless Docker (after checking prerequisites)

1. Create users `aio-ben` and `aio-colleague` with unique credentials and an SSH authentication method.
2. Confirm `grep '^aio-ben:' /etc/subuid /etc/subgid` and the equivalent for `aio-colleague` both show at least 65536 subordinate IDs and don't overlap.
3. Ensure `uidmap` and Docker's rootless extra tools are installed. Ubuntu 24.04 AppArmor user-namespace restrictions are documented by Docker; the Debian package installs the needed profile.
4. Log into each account through SSH, then run `dockerd-rootless-setuptool.sh install --force` (when coexisting system Docker is active).
5. In EACH genuine user shell: `systemctl --user enable --now docker`, `export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock`, `docker info`. Confirm rootless.
6. Check `loginctl enable-linger` on the host to persist rootless daemons across reboot.

The real `XDG_RUNTIME_DIR` is usually `/run/user/$(id -u)`, but do not blindly fake a session if it is missing.

## Run the AIO mastercontainer

Run as `aio-ben`:

```bash
git clone https://github.com/Ben-001-sys/nextcloud-aio-dokploy.git
cd nextcloud-aio-dokploy
cp .env.example .env
# confirm .env has AIO_ADMIN_PORT=8080 AIO_APACHE_PORT=11000 AIO_TALK_PORT=3478
export DOCKER_HOST="unix://$XDG_RUNTIME_DIR/docker.sock"
docker compose config
docker compose up -d
```

As `aio-colleague`, clone into that user's home and set these values in THEIR `.env` before running the same commands:

```dotenv
AIO_ADMIN_PORT=8081
AIO_APACHE_PORT=11001
AIO_TALK_PORT=3479
```

Verify in both sessions that `docker ps` only shows the corresponding rootless user's containers. Do not use `sudo docker compose`.

## AIO administrator interfaces

Use SSH tunnels from your own workstation to the VPS:

```bash
ssh -L 8080:127.0.0.1:8080 aio-ben@VPS_IP
# For the other instance use a separate terminal/connection:
ssh -L 8081:127.0.0.1:8081 aio-colleague@VPS_IP
```

Then access `https://127.0.0.1:8080` and `https://127.0.0.1:8081` separately, accepting only the expected AIO self-signed admin UI certificate after verifying the SSH endpoint.

## Connect Dokploy Traefik

- **Do not create a Dokploy Compose service for either rootless AIO instance.** The system Docker API is separate.
- The AIO Apache children are NOT on `dokploy-network`, because this is a separate Docker engine. `APACHE_ADDITIONAL_NETWORK=dokploy-network` must not be set.
- Use the file-provider example under `traefik/` as a starting template only. It must point to a real **host IP reachable from inside Dokploy Traefik**, with ports 11000 and 11001 for the respective hostname. Determine reachability before enabling routes; `127.0.0.1` inside the Traefik container means Traefik itself, not the VPS host.
- Check the actual mounted Traefik dynamic config directory and existing router definitions; avoid duplicate/conflicting routers.
- Do not bypass AIO's domain validation. Set working DNS and HTTPS routes before completing the AIO wizard.
- Validate forwarding/client IP, TLS, Office collaboration, persistence, backup/restore and optional Talk networking separately.

## Support and reference

- [Official AIO multiple-instance instructions](https://github.com/nextcloud/all-in-one/blob/main/multiple-instances.md)
- [Official AIO rootless instructions](https://github.com/nextcloud/all-in-one/blob/main/docker-rootless.md)
- [Official AIO reverse-proxy instructions](https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md)
- [Docker rootless mode](https://docs.docker.com/engine/security/rootless/)
- [Ubuntu rootless troubleshooting](https://docs.docker.com/engine/security/rootless/troubleshoot/)
