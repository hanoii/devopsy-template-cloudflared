# AGENTS.md

devopsy-template-cloudflared is a Cloudflare Tunnel's connectors for a [devopsy-template-traefik](https://github.com/hanoii/devopsy-template-traefik) server, run with [devopsy-cli](https://github.com/hanoii/devopsy-cli): two `cloudflared` services that connect out to Cloudflare and hand visitors to Traefik's `tunnel` entrypoint, one instance per Cloudflare account. `README.md` is how to use it, kept in step with every change; how a file works is in its comments.

## Layout

- Everything devopsy reads is in `.devopsy/`: `config.yaml` (`project: cloudflared`, `instances: true`, the one secret, the steps, the environment `prod`), `compose.yaml` (`cloudflared-a` and `cloudflared-b` from one anchor, the `tunnel` network), `commands/deploy` (POSIX `sh`, the release, rollback and reload step) and `.env.example` (the variables a target sets).
- No `mnt` and no capabilities: the connectors have no data and no URL of their own.

## Rules

- The image is `latest`, one release kept (`releases: keep: 1`): cloudflared publishes no series tag (only exact versions, dates like `2026.10.5`, `latest` and build ids), and a connector is kept current, never rolled back.
- Each release pulls `latest`, and `deploy` recreates the connectors one at a time when it changed.
- Accepted: what runs changes with no commit, servers released apart run different versions, and a version that never connects fails the release at `cloudflared-a` with no rollback, since a rollback would pull `latest` again.
- The way out of a version that never connects is pinning the last good version and releasing.
- A server nobody releases gets no updates, so the README says to release on a schedule.
- `user:` is set in `compose.yaml`, so a new image's `USER` changes nothing; its entrypoint (`cloudflared --no-autoupdate`) is the image's.
- The connectors are on the `tunnel` network alone, external, with no `default` network: the Traefik template creates `tunnel` with that fixed name, its `tunnel` entrypoint listens there, and it trusts the visitor's IP from that network's subnet alone, so a connector on any other network reaches nothing and is trusted for nothing.
- No `default` network, since nothing on the server needs to reach a connector; the connectors' egress (7844 to Cloudflare) goes through the `tunnel` network's gateway.
- Never a third, shared network name: a project joins another's network as `external`.
- Two services, never `deploy: {replicas: 2}`: Compose recreates every replica of a changed service at once, so a release or a token rotation would drop both connectors together.
- The two services share one YAML anchor (`x-cloudflared`), and `deploy` recreates them one at a time, each up and healthy before the next, so the tunnel always has a connector.
- The two connectors stay identical: a difference between them is a difference between visitors.
- Connected is healthy: the healthcheck is `cloudflared tunnel ready`, and `deploy` runs `up -d --wait` per connector, so a release waits for the connection before the second connector, and a token nobody accepts fails the release before anything else changes (a connector never exits on one, it retries).
- One instance per Cloudflare account (`instances: true`): a tunnel belongs to an account, not to a zone or a server, and a proxied CNAME to `<id>.cfargotunnel.com` works only from a zone of the tunnel's own account.
- A second account is a second target (`cloudflared/<instance>/prod`, its own compose project and `shared/.env`), never a second service or an override block, and the token stays in its account's instance.
- No `devopsy.role`, since several instances hold none and nothing imports from a tunnel.
- No Traefik labels and no `devopsy.import` of the base domain: nothing routes to this project.
- The token is the one setting and the one secret, `PROXY_TUNNEL_TOKEN` (the proxy's contract, `PROXY_*`), required (`${...:?}`, so a release fails before it goes live while it is unset, with its `--vars set` line), listed in `secrets:`; it lets a connector run that tunnel and nothing else.
- Everything about the tunnel but the token (its routes, the hostname) is in the Cloudflare dashboard, where the token is also rotated; `reload: deploy` applies a new one, connectors recreated one at a time.
- The services keep the hardening: the image's own non-root UID (65532, stated in `user:` so an upstream change never changes it silently), `read_only`, `cap_drop: [ALL]` and `no-new-privileges`, nothing published and nothing mounted.
- A capability comes back only with a stated reason (none: a connector only connects out and listens above 1024), and a new service gets the UID, `read_only`, `cap_drop: [ALL]` and `no-new-privileges`.
- No em dashes anywhere (code, docs, commits).

## Connectors

- `TUNNEL_METRICS` is set as a variable, never `--metrics` on the command: `tunnel ready` reads the same variable, so the process and its healthcheck name one address, where a flag would be written twice and could drift.
- Without `TUNNEL_METRICS`, `tunnel ready` fails with `http:///ready: no Host in request URL`.
- The metrics address is on loopback, `127.0.0.1:2000`: only the healthcheck reads it, and the Traefik template's `domains` reports each connector by its Docker health, so nothing on the `tunnel` network reaches it.
- `command: [tunnel, run]` alone: the image's entrypoint is `cloudflared --no-autoupdate` already, and the token comes from `TUNNEL_TOKEN`.
- `stop_grace_period: 40s`: on SIGTERM cloudflared stops taking new requests (the edge sends them to the other connector) and waits for the ones in flight up to its own grace period, 30 s by default, where Docker's default 10 s would cut them on every recreate.
- A value that is no token at all exits at once ("Provided Tunnel token is not valid"), restarted by `restart: unless-stopped`, so the healthcheck never runs and `--wait` fails on the exit.
- A token of the right shape that no tunnel owns keeps the connector running and retrying, `/ready` answering 503 with `readyConnections: 0`, so `--wait` fails once the healthcheck turns unhealthy (its `start_period`, then three failures).
- Either way `deploy` fails at `cloudflared-a`, before touching `cloudflared-b`, which goes on serving on the token it had; the token is the instance's `shared/.env`, not a release's, so the fix is a good token and a reload.
- Without a network path out at all cloudflared exits after its edge discovery fails, and Docker restarts it.
- `tunnel ready` prints nothing on success (exit 0), and on failure the status and the `/ready` JSON (exit 1), so `deploy`'s last line reads the health from `devopsy ps --format '{{.Service}} {{.Health}}'` rather than running `ready` again.
- The image has no shell (distroless), so a healthcheck can only be the binary itself, and `/ready` or `/metrics` is read by hand from a container in the connector's network namespace (`docker run --rm --network container:<id> busybox:1.37 wget -q -O - http://127.0.0.1:2000/ready`).
- `read_only` needs no tmpfs: cloudflared writes nothing to its filesystem.
- quic-go's warning about a small UDP receive buffer (7.5 MB recommended) is a host setting, not this template's: devopsy-server raises it, and a connector works regardless.

## Checks

```sh
cd .devopsy && docker run --rm -v "$PWD:/mnt" -w /mnt koalaman/shellcheck:stable -s sh -x commands/*
PROXY_TUNNEL_TOKEN=dummy docker compose -f .devopsy/compose.yaml config
```

The hardening, on a running environment (`devopsy @<target> exec -T` for one on a server; the image has no shell, so from outside):

```sh
docker inspect "$(devopsy ps -q cloudflared-a)" --format 'User={{.Config.User}} ReadOnly={{.HostConfig.ReadonlyRootfs}} CapDrop={{.HostConfig.CapDrop}} SecOpt={{.HostConfig.SecurityOpt}}'
docker run --rm --network "container:$(devopsy ps -q cloudflared-a)" busybox:1.37 wget -q -O - http://127.0.0.1:2000/ready
```

- A connection that succeeds (`/ready` 200, `--wait` returning) needs a real tunnel's token, so it is checked by hand on a test server with a tunnel of one's own, never a real one.
- With that tunnel: both connectors in the dashboard, a site answering through Cloudflare, a release with a changed pin and a `--reload` with a refreshed token while a request loop against the site sees no error.

## Histories

The root's sections have `internals/rules.history.md`, `internals/connectors.history.md` and `internals/checks.history.md`, read as Workflow's Docs says.
