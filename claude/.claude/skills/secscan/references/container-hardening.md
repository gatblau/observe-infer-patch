# Container and deployment hardening

## Dockerfile

- `USER root` or missing `USER` directive — **medium** (container runs as root).
- `FROM <image>:latest` — **low** (non-reproducible; supply-chain risk on rebuild).
- `ADD http://...` fetching remote content — **medium**. Prefer pinned COPY or multi-stage with verification.
- `ADD` used where `COPY` would suffice — **info** (ADD has extraction side effects).
- `curl | sh` / `wget | bash` inside RUN — **high**.
- Secrets passed via `ARG` and ending up in image history — **high**. Use BuildKit secret mounts.
- `HEALTHCHECK` missing — **info**.
- Package manager cache not cleaned (`apt-get install` without `--no-install-recommends` and `rm -rf /var/lib/apt/lists/*`) — **info** (size, not security).

## docker-compose / k8s

- `privileged: true` — **high**.
- `cap_add: ALL` — **high**.
- Host path mounts from `/` or `/etc` / `/var/run/docker.sock` — **high** (container escape surface).
- `network_mode: host` — **medium**.
- Default network without explicit name in compose — **info**.
- k8s: missing `securityContext.runAsNonRoot: true` — **medium**.
- k8s: missing `readOnlyRootFilesystem: true` — **low**.
- k8s: `hostNetwork`, `hostPID`, `hostIPC` set true — **high**.

## Image provenance

- No image signature verification (cosign / notation) in CI — **low**. Flag as supply-chain gap.
- Pulling from unpinned tag (`:latest`, `:stable`) in deployment manifests — **medium**.

## Secrets in env

- `environment:` blocks with literal secret values — **high**.
- `.env` files referenced but present and tracked — **high**.
