# devopsy-template-cloudflared

The connectors of a [Cloudflare
Tunnel](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/)
for a [devopsy-template-traefik](https://github.com/hanoii/devopsy-template-traefik)
server, released with [devopsy](https://github.com/hanoii/devopsy-cli).
Two `cloudflared` services connect out to Cloudflare and visitors come in
through them, so the server publishes no port for its sites, only SSH.
There is one instance per Cloudflare account.

## Using this template

This template is meant to be copied and changed. Start a project from it
on GitHub ("Use this template"), or clone it and keep this repository as a
remote (`upstream`) to pull its changes when you choose. Your copy and your
servers change only when you release.

## Setting up a tunnel

1. Release devopsy-template-traefik on the server. Its [Behind a
   Cloudflare
   Tunnel](https://github.com/hanoii/devopsy-template-traefik/blob/main/docs/tunnel.md)
   covers the base domain, the DNS record, the SSL mode and what the edge
   covers. Traefik creates the `tunnel` network, which this project joins,
   and listens on it for the connectors. A release here fails while that
   network does not exist.
2. In the Cloudflare dashboard (Networking, Tunnels), create a tunnel of
   the Cloudflared type and name it after the server. Copy its token from
   the connector install command (the value after `--token`), without
   running the command. The dashboard then waits for a connector.
3. Release this project, as one instance named after the account (`own`
   here). The server goes in `.devopsy/.env` (gitignored) or in the
   address, and the token in the instance's `shared/.env`:

   ```sh
   echo DEVOPSY_SERVER=devopsy@203.0.113.10 >> .devopsy/.env
   devopsy @vm1:own/prod --vars set PROXY_TUNNEL_TOKEN   # asks for it, or reads it from your environment
   devopsy @vm1:own/prod --release                       # runs deploy: both connectors up and connected
   devopsy @vm1:own/prod --status
   devopsy @vm1:own/prod logs -f
   ```

4. Once the dashboard shows the two connectors, publish one application
   route: hostname `*.<base domain>` (the subdomain `*`), service
   `http://traefik.tunnel:8081`. Every request goes there, to Traefik's
   `tunnel` entrypoint by its network-scoped name, and Traefik routes it
   to the site by its host.

The sites answer at their wildcard URLs through Cloudflare. `devopsy
@<server>-traefik domains` (the Traefik template) reports the hosts, and
the connectors it sees on the `tunnel` network.

## The token

The token only runs this tunnel. It gives no access to DNS, zones or
account settings, but anyone holding it could connect their own origin to
your tunnel, so treat it as a secret.

It lives in the instance's `shared/.env` on the server, out of git. Set it
with `--vars set`, which sends the value over SSH's stdin rather than as
an argument. `config.yaml` names it in `secrets:`, so devopsy masks it in
`--vars`, `--env` and its messages.

To rotate it, refresh the token in the dashboard (the tunnel's
configuration), then set and apply the new one:

```sh
devopsy @vm1:own/prod --vars set PROXY_TUNNEL_TOKEN
devopsy @vm1:own/prod --reload
```

`deploy` recreates the connectors one at a time, so the tunnel keeps
serving through the rotation, and a connector still on the old token
keeps working until it is recreated. A token no tunnel accepts fails at
the first connector ([How deploy works](#how-deploy-works)). The token is
set on the instance, so no rollback brings the old one back: the tunnel
runs on the second connector alone until a good token is set and applied.
`devopsy @vm1:own/prod logs cloudflared-a` says why the first one failed.

## One instance per account

A tunnel belongs to a Cloudflare account, so this project runs one
instance per account whose zones route through the server, with that
account's tunnel and token. Every account publishes its routes to the
same `http://traefik.tunnel:8081`:

```sh
devopsy @vm1:own/prod --release    # your account: the server's base domain, your zones
devopsy @vm1:acme/prod --release   # a client's account: their zones, through their edge
```

An instance is its own compose project on the server
(`cloudflared-own-prod`, in `cloudflared/own/prod` under its release
root), with its own `shared/.env`, so a token never leaves its account's
instance. Most servers need one, since they host one client whose account
holds every zone. The Traefik template's [A tunnel per Cloudflare
account](https://github.com/hanoii/devopsy-template-traefik/blob/main/docs/tunnel.md#a-tunnel-per-cloudflare-account)
explains why a zone of another account cannot use the tunnel, when a
second instance is needed, and the alternatives.

A tunnel also belongs to one server, so never release the same token on
two servers. Cloudflare accepts it, as replicas of one tunnel, but sends
each request to whichever connector it picks, and you cannot route by
hostname. A connector reaches only its own server's Traefik, so a site on
one server would answer from the other with a 404, at random. Give every
server its own tunnel, and when a site moves to another server, move its
hostname to that server's tunnel.

## How deploy works

`compose.yaml` defines `cloudflared-a` and `cloudflared-b` from one block,
on the same token, so the tunnel has two connectors, as Cloudflare
recommends. They are two services rather than two replicas of one, since
compose recreates every replica of a changed service at once.

`deploy` starts each connector and waits until it is connected before
starting the other, so a release (a new `cloudflared`) or a reload (a new
token) always leaves a connector serving. Each connector's healthcheck is
`cloudflared tunnel ready`, which asks its own metrics endpoint (`/ready`,
on the container's loopback, port 2000) and fails until a connection to
the edge is up, and `up --wait` waits for that healthcheck. A token or a
version that never connects therefore fails the release at the first
connector, and the second keeps serving on the token and version it had.

## Network and hardening

The connectors open connections to Cloudflare on port 7844 (QUIC over
UDP, else HTTP/2 over TCP) and receive visitors through them. They join
only Traefik's `tunnel` network, and reach Traefik and the Internet
through it; nothing needs to reach them. Traefik trusts the visitor's IP
(`CF-Connecting-IP`) from that network's subnet, so sites see their
visitors' IPs.

`cloudflared` runs as the image's own UID 65532, with a read-only root
filesystem, no capabilities and `no-new-privileges`. The image has no
shell.

`cloudflared` logs a warning when the host's UDP receive buffer is small
for QUIC (`net.core.rmem_max`). It still works, and a `sysctl` raising
the buffer to 7500000 silences the warning; devopsy-server's setup sets
it.

## Updates

`compose.yaml` runs `cloudflare/cloudflared:latest`. cloudflared publishes
no series tag to float on (its versions are dates, like `2026.10.5`), and
a connector is only ever upgraded, so `config.yaml` keeps one release.

Every release pulls `latest` and, when it changed, recreates the
connectors one at a time; a `--reload` pulls nothing. A server you never
release keeps the version it has, so release on a schedule (monthly, from
CI or a cron), or whenever you release the server's other projects.
Cloudflare supports a version for about a year.

A new version that never connects fails the release at the first
connector ([How deploy works](#how-deploy-works)). There is no rollback to
go back to, so pin the last good version in `compose.yaml` and release.

Nothing in git records which version a server runs, and servers released
at different times differ. A connector's log names its version: `devopsy
@vm1:own/prod logs cloudflared-a`, or after a failed release,
`cloudflared-b` for the last good one.

## Destroying

`devopsy @vm1:own/prod --destroy` takes the connectors down and removes
the instance. Delete the tunnel in the dashboard too, or keep it for
another server.

## Locally

With a local devopsy-template-traefik running (it creates the `tunnel`
network) and a token in `.devopsy/.env`, the connectors run locally too:

```sh
devopsy --release
```

Use a tunnel made for local use. Cloudflare spreads a tunnel's visitors
over all its connectors, so a server's token here would send part of that
server's visitors to your local Traefik.

## License

MIT, so you can start your own project from it. See [LICENSE](LICENSE).
