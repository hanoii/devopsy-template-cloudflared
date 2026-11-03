History, not rules: AGENTS.md's Connectors win where they differ.

- `--metrics` is an option of the `tunnel` command, shared by `run` and `ready`, with `TUNNEL_METRICS` as its variable; a flag would be written twice (`tunnel --metrics ... run` and `tunnel --metrics ... ready`). "no Host in request URL" without it was verified.
- A token of the right shape that no tunnel owns logs "Register tunnel error from server side", backing off; `--wait` failed after 52 s, measured. Without a network path out was tried with `--network none`.

Verified on `cloudflare/cloudflared:2026.10.5` (2026-10-28), with real Docker (OrbStack, arm64; the tag has amd64 and arm64 images):

- `docker image inspect`: `User=65532:65532`, `Entrypoint=["cloudflared", "--no-autoupdate"]`, `Cmd=["version"]`, `SSL_CERT_FILE` set, no shell (`--entrypoint /bin/sh` fails: no such file).
- `cloudflared tunnel ready` exists ("Call /ready endpoint and return proper exit code"); `--metrics` is a `tunnel` command option with `[$TUNNEL_METRICS]`, listed for `ready` and `run` alike; `--grace-period` (default 30 s) is a `tunnel` option too.
- Run as `--user 65532:65532 --read-only --cap-drop ALL --security-opt no-new-privileges:true` with `TUNNEL_METRICS=0.0.0.0:2000` and a shaped token: starts, "Starting metrics server on [::]:2000/metrics", reaches the edge and is refused ("Failed to get tunnel"), keeps retrying, still running after 30 s with no restart; `docker exec <c> cloudflared tunnel ready` prints the 503 and `{"status":503,"readyConnections":0,...}`, exit 1; with `TUNNEL_METRICS` unset, "no Host in request URL"; a wrong address, "connection refused", exit 1. `docker diff` shows nothing of cloudflared's (only OrbStack's injected CA certificate), so `read_only` needs no tmpfs.
- `deploy` locally with the shaped token (the `tunnel` network created by hand): `up -d --wait cloudflared-a` failed "container ... is unhealthy" after 52 s, `cloudflared-b` never created; the container's `User`, `ReadonlyRootfs`, `CapDrop=[ALL]`, `SecurityOpt`, `StopTimeout=40` and its one network, `tunnel`, as written.
