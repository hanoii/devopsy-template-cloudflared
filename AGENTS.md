# AGENTS.md

devopsy-template-cloudflared is a Cloudflare Tunnel's connectors for a
[devopsy-template-traefik](https://github.com/hanoii/devopsy-template-traefik)
server, run with [devopsy-cli](https://github.com/hanoii/devopsy-cli):
two `cloudflared` services that connect out to Cloudflare and hand
visitors to Traefik's `tunnel` entrypoint, one instance per Cloudflare
account. `README.md` is how to use it; keep it in step with every change.

This file holds the rules, the decisions with their reasons, and the
findings a later session must not have to rediscover. How a file works
is in its comments.

## Layout

Everything devopsy reads is in `.devopsy/`: `config.yaml` (`project:
cloudflared`, `instances: true`, the one secret, the steps, the
environment `prod`), `compose.yaml` (`cloudflared-a` and `cloudflared-b`
from one anchor, the `tunnel` network), `commands/deploy` (POSIX `sh`,
the release, rollback and reload step) and `.env.example` (the variables
a target sets). No `mnt`, no capabilities: the connectors have no data
and no URL of their own.

## Rules

- **Exact pin, bumped by hand:** `cloudflare/cloudflared:<version>`,
  the version a date (`2026.10.5`). Why: cloudflared publishes no series
  tag (only exact versions, `latest` and build ids), and `latest` would
  change what runs with no commit. A bump is an edit and a release, which
  recreates the connectors one at a time; a rollback takes the pin back
  (devopsy-cli's README, "Choosing tags"). Check the image's `USER` and
  `ENTRYPOINT` again when bumping (below), and read the release notes.
- **The `tunnel` network alone, external, and no `default` network.**
  The Traefik template creates it with that fixed name (as it does
  `traefik`), its `tunnel` entrypoint listens on it, and it trusts the
  visitor's IP from that network's subnet alone: a connector anywhere
  else would reach nothing or be trusted for nothing. No `default`
  network because nothing on the server needs to reach a connector, and
  every network a project owns is one more thing to name and check. The
  connectors' egress (7844 to Cloudflare) goes through that network's
  gateway. Never a third, shared name: a project joins another's
  network as `external`.
- **Two services, never `deploy: {replicas: 2}`.** Compose recreates
  every replica of a changed service at once, so a release or a token
  rotation dropped both connectors together, and Cloudflare answered
  1033 until one reconnected. Two services sharing one YAML anchor
  (`x-cloudflared`: anchors work within a file) are recreated one at a
  time by `deploy`, each up and healthy before the next, so the tunnel
  always has a connector. Keep them identical: a difference between the
  two is a difference between visitors.
- **Connected is healthy:** the healthcheck is `cloudflared tunnel
  ready`, and `deploy` uses `up -d --wait` per connector. Why: without
  it `--wait` only waits for running, and the gap is the connection
  time, during which a release would go on to the second connector and
  the tunnel would have none; and a token nobody accepts would leave a
  connector retrying forever (it never exits on it: verified) where it
  should fail the release before anything else changes.
- **One instance per Cloudflare account** (`instances: true`): a
  tunnel belongs to an account, not to a zone or a server, and a proxied
  CNAME to `<id>.cfargotunnel.com` only works from a zone of the
  tunnel's own account (from another, error 1033, as when no connector
  is up: verified). devopsy runs several
  instances of one project side by side (`cloudflared/<instance>/prod`,
  each its own compose project and `shared/.env`), so a second account is
  a second target, not a second service or an override block; the token
  stays in its account's instance. No `devopsy.role`: several instances
  hold none, and nothing imports from a tunnel. No Traefik labels and no
  `devopsy.import` of the base domain: nothing routes to this project.
- **The token is the one setting and the one secret,** `PROXY_TUNNEL_TOKEN`
  (the proxy's contract, `PROXY_*`), required (`${...:?}`, so a release
  fails before it goes live while it is unset, with its `--vars set`
  line), listed in `secrets:`. It only lets a connector run that tunnel;
  everything else about the tunnel (its routes, the hostname) is in the
  Cloudflare dashboard, where the token is also rotated. `reload: deploy`
  applies a new one, connectors recreated one at a time.
- **Keep the hardening:** the image's own non-root UID (65532, stated in
  `user:` so an upstream change never changes it silently), `read_only`,
  `cap_drop: [ALL]`, `no-new-privileges`, nothing published, nothing
  mounted; a capability comes back only with a stated reason (none: it
  only connects out and listens above 1024). A new service gets all four.
- **Names in examples and docs are generic** (`vm1`, `own`, `acme`,
  `example.com`, `203.0.113.10`): never a real server, account, client,
  domain or person.
- **No em dashes anywhere** (code, docs, commits), and any word with an
  `@` in a commit message or a doc goes in backticks (`@vm1:own/prod`):
  GitHub links a bare `@name` to that user.

## Decisions and gotchas

- **`TUNNEL_METRICS` as a variable, not `--metrics` on the command.**
  `--metrics` is an option of the `tunnel` command, shared by `run` and
  `ready`, with `TUNNEL_METRICS` as its variable; `tunnel ready` reads
  the same variable, so one value in `environment:` makes the process
  and its healthcheck name one address, where a flag would be written
  twice (`tunnel --metrics ... run` and `tunnel --metrics ... ready`) and
  could drift. Without it `ready` fails with `http:///ready: no Host in
  request URL` (verified). On loopback, `127.0.0.1:2000`: only the
  healthcheck reads it, and the Traefik template's `domains` reports each
  connector by its Docker health, so nothing on the `tunnel` network
  reaches it.
- **`command: [tunnel, run]` alone:** the image's entrypoint is
  `cloudflared --no-autoupdate` already, and the token comes from
  `TUNNEL_TOKEN`, so the command names only the subcommand.
- **`stop_grace_period: 40s`:** on SIGTERM cloudflared stops taking new
  requests (the edge sends them to the other connector) and waits for
  the ones in flight up to its own grace period, 30 s by default; Docker
  kills at 10 s otherwise, cutting those requests on every recreate.
- **A wrong token.** One that is not even a token (`not-a-token`) exits
  at once, "Provided Tunnel token is not valid", and `restart:
  unless-stopped` restarts it: the healthcheck never runs, `--wait`
  fails on the exit. One with a token's shape that no tunnel owns keeps
  running and retrying ("Register tunnel error from server side", backing
  off), `/ready` answering 503 with `readyConnections: 0`: `--wait`
  fails once the healthcheck turns unhealthy (its `start_period`, then
  three failures: 52 s measured). Either way `deploy` fails at
  `cloudflared-a`, before touching `cloudflared-b`, which goes on
  serving on the token it had. The token is the instance's
  `shared/.env`, not a release's, so the release before fails the same
  way when a release goes back to it: the fix is a good token and a
  reload. Without a network path out at
  all (`--network none`) cloudflared exits after its edge discovery
  fails, and Docker restarts it.
- **`tunnel ready` prints nothing on success** (exit 0), and on failure
  the status and the `/ready` JSON (exit 1). So `deploy`'s last line
  reads the health from `devopsy ps --format '{{.Service}}
  {{.Health}}'` rather than running `ready` again.
- **The image has no shell** (distroless): no `docker exec ... sh`, and
  a healthcheck can only be the binary itself. To read `/ready` or
  `/metrics` by hand, run a container with `wget` in the connector's
  network namespace (`docker run --rm --network container:<id>
  busybox:1.37 wget -q -O - http://127.0.0.1:2000/ready`).
- **`net.core.rmem_max`:** quic-go logs a warning when the host's UDP
  receive buffer is small (7.5 MB recommended); it works regardless.
  A host setting, not this template's: devopsy-server raises it.

### Verified on `cloudflare/cloudflared:2026.10.5` (2026-10-28)

With real Docker (OrbStack, arm64; the tag has amd64 and arm64 images):

- `docker image inspect`: `User=65532:65532`, `Entrypoint=["cloudflared",
  "--no-autoupdate"]`, `Cmd=["version"]`, `SSL_CERT_FILE` set, no shell
  (`--entrypoint /bin/sh` fails: no such file).
- `cloudflared tunnel ready` exists ("Call /ready endpoint and return
  proper exit code"); `--metrics` is a `tunnel` command option with
  `[$TUNNEL_METRICS]`, listed for `ready` and `run` alike;
  `--grace-period` (default 30 s) is a `tunnel` option too.
- Run as `--user 65532:65532 --read-only --cap-drop ALL --security-opt
  no-new-privileges:true` with `TUNNEL_METRICS=0.0.0.0:2000` and a
  shaped token: starts, "Starting metrics server on [::]:2000/metrics",
  reaches the edge and is refused ("Failed to get tunnel"), keeps
  retrying, still running after 30 s with no restart; `docker exec
  <c> cloudflared tunnel ready` prints the 503 and
  `{"status":503,"readyConnections":0,...}`, exit 1; with
  `TUNNEL_METRICS` unset, "no Host in request URL"; a wrong address,
  "connection refused", exit 1. `docker diff` shows nothing of
  cloudflared's (only OrbStack's injected CA certificate), so
  `read_only` needs no tmpfs.
- `deploy` locally with the shaped token (the `tunnel` network created
  by hand): `up -d --wait cloudflared-a` failed "container ... is
  unhealthy" after 52 s, `cloudflared-b` never created; the container's
  `User`, `ReadonlyRootfs`, `CapDrop=[ALL]`, `SecurityOpt`,
  `StopTimeout=40` and its one network, `tunnel`, as written.

A connection that succeeds (`/ready` 200, `--wait` returning) needs a
real tunnel's token: checked by hand on a test server with a tunnel of
your own, never a real one.

## Checks

```sh
cd .devopsy && docker run --rm -v "$PWD:/mnt" -w /mnt koalaman/shellcheck:stable -s sh -x commands/*
PROXY_TUNNEL_TOKEN=dummy docker compose -f .devopsy/compose.yaml config
```

The hardening, on a running environment (`devopsy @<target> exec -T`
for one on a server; the image has no shell, so from outside):

```sh
docker inspect "$(devopsy ps -q cloudflared-a)" --format 'User={{.Config.User}} ReadOnly={{.HostConfig.ReadonlyRootfs}} CapDrop={{.HostConfig.CapDrop}} SecOpt={{.HostConfig.SecurityOpt}}'
docker run --rm --network "container:$(devopsy ps -q cloudflared-a)" busybox:1.37 wget -q -O - http://127.0.0.1:2000/ready
```

Then, with a real tunnel on a test server: both connectors in the
dashboard, a site answering through Cloudflare, a release with a changed
pin and a `--reload` with a refreshed token while a request loop against
the site sees no error.

## Commits

Conventional commits (`feat:`, `fix:`, `docs:`, `build:`...), with no
attribution trailers. Any word with an `@` in backticks.
