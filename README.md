# Nextcloud AIO on Dokploy (separate experimental deployment)

This is a **starter configuration** for Nextcloud All-in-One (AIO) behind an existing Dokploy Traefik reverse proxy. It is **not** a standard Nextcloud Compose deployment or a one-click Dokploy template. It has **not been tested on the VPS**.

## Architecture

The `compose.yaml` starts only the AIO **mastercontainer**. After setup, AIO creates and manages its own sibling containers for Nextcloud, database, Redis, and optional Collabora/Talk services. Do **not** add the standard Nextcloud app, Postgres, Redis, and Cron services from the existing deployment here.

The mastercontainer requires the fixed container name and named volume. It also requires access to the Docker API (via `/var/run/docker.sock`), which grants substantial control over the **entire host**, including Dokploy's other services. Use a separate VM or a suitable isolated Docker environment for stronger separation.

## Critical prerequisites

1. **Use a new domain** (for example, `aio.example.com`) with an A record pointing to the host. Never reuse `nextcloud-ben.benuel.dedyn.io`.
2. Check that no AIO installation already exists on this Docker engine using `docker ps -a --filter name=nextcloud-aio`. The normal fixed naming convention prevents running two independent AIO mastercontainers on one Docker engine. Multiple AIO installations require a supported separate-VM/rootless-Docker approach.
3. Verify `dokploy-network` exists, `dokploy-traefik` belongs to it, and the network is attachable for AIO's standalone Apache child: `docker network inspect dokploy-network --format 'driver={{.Driver}} scope={{.Scope}} attachable={{.Attachable}}'`. If `attachable=false`, stop and revise the networking approach before starting AIO.
4. Check that ports 8080, 11000 and the chosen Talk port (default example: 3479) are not occupied. Only the AIO admin UI port 8080 is bound to host loopback. The AIO Apache port 11000 is configured with `APACHE_IP_BINDING=0.0.0.0` so Traefik can reach it through the shared Docker network; restrict **public inbound TCP 11000** using the VPS provider or upstream firewall. Docker-published ports may bypass some host firewall rules. Optional Talk services can require externally reachable ports.
5. Confirm CPU, RAM, storage, backups, firewall, and SSH access are sufficient before deployment. Optional services increase resource requirements.

## Deploy AIO mastercontainer

1. In Dokploy create a **new project** and **Compose** service. Select this repository, branch `main`, and Compose path `./compose.yaml`.
2. Set `AIO_TALK_PORT=3479` if that host port is available, or use another free port. Do not commit passwords.
3. Do **not** create a normal Dokploy app domain targeting mastercontainer port 8080 or route the Nextcloud domain directly to the AIO administrator UI.
4. Deploy the mastercontainer. Do not proceed if another AIO already uses its fixed names or volumes.
5. From your workstation establish a trusted SSH tunnel:

   ```bash
   ssh -L 8080:127.0.0.1:8080 root@YOUR_VPS_IP
   ```

6. Open `https://127.0.0.1:8080` locally. The AIO administrative UI has its own certificate; verify you have reached your own host.
7. Configure a **new, separate** hostname before completing the AIO wizard.

## Configure Traefik routing

This setup uses Traefik's file provider because AIO's **child** Apache container, not the mastercontainer, hosts Nextcloud.

1. On the VPS inspect the Traefik container mounts and ensure the dynamic file provider is enabled:

   ```bash
   docker inspect dokploy-traefik --format '{{json .Mounts}}'
   docker network inspect dokploy-network
   ```

2. Copy `traefik/nextcloud-aio.example.yml` to the **actual mounted dynamic-config directory** and replace `aio.example.com` with the new AIO hostname.
3. The example uses the observed entrypoints `web`, `websecure`, and resolver `letsencrypt`; verify these still match your Traefik instance.
4. The example routes to `http://nextcloud-aio-apache:11000` on `dokploy-network`. AIO connects its Apache child through `APACHE_ADDITIONAL_NETWORK`.
5. Do not create a competing Dokploy domain router for this hostname. The dynamic file is an **example only**; it is not applied automatically by Dokploy from this repository.
6. Complete the AIO setup wizard, enabling only the optional components you need. Validate HTTPS, Office editing, file persistence, and Talk calling separately.

**Avoid disabling AIO domain validation just to bypass a proxy/DNS failure.** Investigate DNS, Traefik routing, container networks, and certificates first.

## Operational notes

- AIO manages its own optional services, upgrades and backups in its dedicated admin interface, not fully through Dokploy.
- Verify off-server backups and restoration. Never delete volumes as a routine redeployment step.
- HSTS is configured for the dedicated AIO hostname only; no subdomain inheritance or preload.
- The source repository may be public only because it contains **no credentials**. Private visibility is recommended for operational documentation.
- Do not run multiple ordinary AIO mastercontainers with identical fixed names on the same Docker engine.

## Official documentation

- [Nextcloud AIO](https://github.com/nextcloud/all-in-one)
- [AIO Compose reference](https://github.com/nextcloud/all-in-one/blob/main/compose.yaml)
- [AIO reverse proxy setup](https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md)
- [Multiple AIO instances](https://github.com/nextcloud/all-in-one/blob/main/multiple-instances.md)
