# devopsy-template-cloudflared

A [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/)'s
connectors for a [devopsy-template-traefik](https://github.com/hanoii/devopsy-template-traefik)
server, released with [devopsy](https://github.com/hanoii/devopsy-cli): two
`cloudflared` services connect out to Cloudflare, visitors come in through
them, and the server publishes no port for its sites, only SSH. One
instance per Cloudflare account.

## Using this template

A starting point to own, not a dependency: start a project from it on
GitHub ("Use this template"), or clone it and keep this repository as a
remote (`upstream`) to pull its changes when you choose. Nothing updates
your copy, or what runs on your servers, but your own release.

## Two releases

A tunneled server is the Traefik template released first, then this one.

1. **Traefik.** Release devopsy-template-traefik on the server (its README,
   ["Behind a Cloudflare Tunnel"](https://github.com/hanoii/devopsy-template-traefik#behind-a-cloudflare-tunnel):
   the base domain, the DNS record, the SSL mode, what the edge covers).
   It creates the `tunnel` network, which this project joins, and listens
   on it for the connectors: a release here fails while that network does
   not exist.
2. **The tunnel, in the Cloudflare dashboard** (Networking, Tunnels):
   create a tunnel of the Cloudflared type and name it after the server;
   copy its token from the connector install command (the value after
   `--token`), without running the command. The dashboard then waits for
   a connector: release this project (step 3), and once it shows the two
   connectors, publish one application route, hostname
   `*.<base domain>` (the subdomain `*`), service `http://traefik.tunnel:8081`.
   That is where every request goes: Traefik's `tunnel` entrypoint, by
   its network-scoped name, which then routes it to the site by host.
3. **This project**, as one instance of it, named after the account
   (`own` here): the server in `.devopsy/.env` (gitignored) or the
   address, the token in the instance's `shared/.env`, then the release.

```sh
echo DEVOPSY_SERVER=devopsy@203.0.113.10 >> .devopsy/.env
devopsy @vm1:own/prod --vars set PROXY_TUNNEL_TOKEN   # asks for it, or reads it from your environment
devopsy @vm1:own/prod --release                       # runs deploy: both connectors up and connected
devopsy @vm1:own/prod --status
devopsy @vm1:own/prod logs -f
```

The dashboard shows the two connectors on the tunnel once the release is
done, and the sites answer at their wildcard URLs through Cloudflare.
`devopsy @<server>-traefik domains` (the Traefik template) reports the
hosts and the connectors it sees on the tunnel network.

## The token

- **What it allows:** running that tunnel, and nothing else: a connector
  with it serves the tunnel's routes; it reads no DNS, no zone, no account
  setting. Someone holding it could connect their own origin to your
  tunnel, so keep it where secrets live.
- **Where it lives:** the instance's `shared/.env` on the server, set with
  `--vars set` (the value travels over SSH's stdin, never as an argument)
  and never in git: `config.yaml` names it in `secrets:`, so devopsy
  masks it in `--vars`, `--env` and its messages.
- **Rotating it:** refresh the token in the dashboard (the tunnel's
  configuration), then set and apply the new one. `deploy` recreates
  the connectors one at a time, each up and connected before the next:
  the tunnel keeps serving through the rotation, and connectors running on
  the old token keep working until they are recreated.

```sh
devopsy @vm1:own/prod --vars set PROXY_TUNNEL_TOKEN
devopsy @vm1:own/prod --reload
```

A token no tunnel accepts fails the deploy at the first connector, which
never gets a connection (its healthcheck), while the second still serves
on the token it had. The token is the instance's, not a release's, so
the tunnel runs on the second connector alone until a good token is set
and applied (`--vars set`, then `--reload`). `devopsy @vm1:own/prod logs
cloudflared-a` says why.

## One instance per account

A tunnel belongs to a Cloudflare account, not to a zone or a server: any
zone of that account points a proxied CNAME at it
(`<tunnel id>.cfargotunnel.com`), and a zone of another account cannot
(Cloudflare answers error 1033, as when no connector is up). So this project runs one instance per
account whose zones route through the server, each with its own tunnel
and token, and each account publishes its own routes to the same
`http://traefik.tunnel:8081`:

```sh
devopsy @vm1:own/prod --release    # your account: the server's base domain, your zones
devopsy @vm1:acme/prod --release   # a client's account: their zones, through their edge
```

Each instance is its own compose project on the server
(`cloudflared-own-prod`, in `cloudflared/own/prod` under its release
root), with its own `shared/.env`, so a token never leaves its account's
instance. Usually one is enough: the server serves one client, whose
account holds every zone. The Traefik template's README says when a
second instance is the answer and what the alternatives are.

A tunnel is one server's, too: never release the same token on two
servers. Cloudflare accepts it, as replicas of one tunnel, but sends
each request to whichever connector it picks, with no say per hostname,
and each connector reaches its own server's Traefik: a site on one
server would answer from the other with a 404, at random. Each server
gets its own tunnel, and moving a site to another server moves its
hostname to that server's tunnel.

## How it works

- **Two connectors, one at a time.** `compose.yaml` defines
  `cloudflared-a` and `cloudflared-b` from one block, on the same token:
  the tunnel has two connectors, as Cloudflare recommends, and `deploy`
  brings each up and connected before touching the other, so a release
  (a new `cloudflared`) or a reload (a new token) never leaves the tunnel
  with none. Two services and not two replicas of one: compose recreates
  every replica of a changed service at once.
- **Connected means healthy.** Each connector's healthcheck is
  `cloudflared tunnel ready`, which asks its own metrics endpoint
  (`/ready`, on the container's loopback, port 2000)
  and fails until a connection to the edge is up, so `up --wait` waits
  for a connector that serves, and a token nobody accepts fails the
  release before the second connector is touched.
- **Outbound only.** The connectors open connections to Cloudflare on
  port 7844 (QUIC over UDP, else HTTP/2 over TCP) and receive visitors
  through them: nothing listens on the server for the sites. They join
  Traefik's `tunnel` network alone and have none of their own: they reach
  Traefik and the Internet through it, and nothing needs to reach them.
  Traefik trusts the visitor's IP (`CF-Connecting-IP`) from that
  network's subnet, so sites see visitors, not the connector.
- **Hardening.** `cloudflared` runs as the image's own UID 65532, with a
  read-only root filesystem, no capabilities and `no-new-privileges`; the
  image has no shell.
- **Destroying.** `devopsy @vm1:own/prod --destroy` takes the connectors
  down and removes the instance; delete the tunnel in the dashboard too,
  or keep it for another server.

## Updates

`compose.yaml` runs `cloudflare/cloudflared:latest`: cloudflared publishes
no series tag to float on (its versions are dates, like `2026.10.5`), and a
connector is something to keep current, never to roll back. So
`config.yaml` keeps one release.

- **A release updates it.** Each release pulls `latest` and, when it
  changed, recreates the connectors one at a time; a `--reload` pulls
  nothing. A server you never release keeps the version it has, so
  release on a schedule (monthly, from CI or a cron) or whenever you
  release the server's other projects. Cloudflare supports a version for
  about a year.
- **A version that never connects fails the release at the first
  connector,** the second still serving on the version it had: pin the
  last good version in `compose.yaml` (the second connector's log,
  `devopsy @vm1:own/prod logs cloudflared-b`, names the version it runs)
  and release. There
  is no rollback to go back to.
- **Two servers released at different times** run different versions,
  which nothing in git records; `devopsy @vm1:own/prod logs
  cloudflared-a` shows which.

`cloudflared` logs a warning when the host's UDP receive buffer is small
for QUIC (`net.core.rmem_max`); it works all the same, and a `sysctl`
raising it to 7500000 silences it (devopsy-server's README).

## Locally

With a local devopsy-template-traefik running (it creates the `tunnel`
network) and a token in `.devopsy/.env`, the connectors run locally
too. Use a tunnel of its own, never a server's: Cloudflare spreads a
tunnel's visitors over all its connectors, so a server's token here
would send part of its visitors to your local Traefik.

```sh
devopsy --release
```

## License

MIT, so you can start your own project from it. See [LICENSE](LICENSE).
