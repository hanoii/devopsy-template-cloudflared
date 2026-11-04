# AGENTS.md

devopsy-template-cloudflared is a Cloudflare Tunnel's connectors for a [devopsy-template-traefik](https://github.com/hanoii/devopsy-template-traefik) server, run with [devopsy-cli](https://github.com/hanoii/devopsy-cli): two `cloudflared` services that connect out to Cloudflare and hand visitors to Traefik's `tunnel` entrypoint, one instance per Cloudflare account. `README.md` is how to use it, kept in step with every change; how a file works is in its comments.

## Workflow

### Git

- Conventional commits; the subject about 50 characters, 72 at most, the reason in the body: release notes are commit subjects, and tools cut longer ones.
- A commit made with an AI agent ends with one line naming it, `Assisted-by: <agent>:<model> [<effort>]` (for example `Assisted-by: claude-code:claude-opus-5-5 high`), never a session link (for example `Claude-Session`) or `Co-Authored-By`, even when a tool asks for one: GitHub shows a co-author as a person.
- Any word with an `@` in a commit message or a doc goes in backticks (for example `@prod`, `deploy@host`): GitHub links a bare `@name` to that user.
- Push and tag only when asked: a push publishes, a tag releases.
- A change of one commit (for example a quick fix) goes straight onto `main`, in the main checkout.
- Any larger task gets one branch, `<type>/<slug>`, its type what it brings to `main` whatever its commits' types (for example `feat/snapshots`, `docs/readme`).
- All of a task's work goes on its branch, reviews' failing tests and their fixes included; never a branch off a task's branch.
- A finished branch is rebased onto `main`, merged with `--no-ff` and deleted.
- A partial merge, only when asked, is rebased and merged the same way and keeps the branch and its worktree, rebased at once onto the merge for the rest of the task.
- A branch is worked in a worktree of its own, `.worktrees/<slug>` (`git worktree add .worktrees/<slug> -b <type>/<slug>`), removed once merged: inside the repository, a worktree is within the directories the agent may use.
- The main checkout stays on `main`.
- A worktree of a devopsy project run beside its main checkout gets its own `project:` (for example `whoami-<slug>`) in `.devopsy/config.local.yaml`: otherwise both are one local compose project, sharing containers and volumes.
- `.worktrees/` is listed in `.gitignore` and in the ignore file of any Docker build context that holds it: a `docker build` would otherwise send every worktree in its context.
- A session in a worktree starts by rebasing it onto `main`: the work adapts to what changed while it is still cheap.

### Tests and checks

- Every change carries its test, a fix's failing without it, a feature's checking what it promises, and a behavior a change touches with no test gets one then: a test is what keeps a fix fixed.
- A fix's test is run against the code before the fix, and the summary says it failed there.
- Unit tests first; an end-to-end test only for what needs a real system (for example Docker, sshd, the network, a terminal or timing).
- One test per behavior: extend the test that covers it before adding another, and never test one thing at both levels.
- The suites stay fast, locally and in CI, since every change waits on them: a new test runs in parallel with the others where it can, polls rather than sleeps, and reuses the setups that exist.
- A test made faster checks the same things, never fewer: a speed-up that drops a case or an assertion is a regression.
- A check or a refusal is only for a mistake any release can make, a setting a tool would otherwise take without a word; none is added for one upgrade only (for example a refusal of an old name after a rename): an upgrade's own steps cover those, and old names left over are ignored.
- Until 1.0.0, no tests of specific older versions and no handling of mixed versions (for example an operator and a server on different devopsy): a new behavior an older server lacks refuses before anything changes, and that is all.

### Refactors

- Before writing code for one case, look for the same pattern elsewhere (for example a message, a check, an interrupt) and propose one helper for all, each site adding only what is its own: variants drift apart, and a fix then lands in one of them only.
- A design that makes a change awkward is proposed for change, never worked around: workarounds pile up where the design is wrong.
- When a change makes the code simpler (less code, fewer external calls), the tests' fakes are extended to match, never the heavier code kept or a test weakened to pass.
- A refactor reaching beyond the task is agreed before it is made.

### Reviews

- A review round is two independent reviewers, one on Opus and one on Fable, on the task's branch in its worktree.
- A review brief lists what earlier rounds found and states Tests and checks' two limits, mixed versions and lean checks: findings of those kinds are not fixed.
- A review brief also asks for the same pattern written twice and for workarounds of the design (Refactors).
- A reviewer writes what it tests into the repository as `test:` commits on the task's branch, a failing test that proves a bug kept with a comment naming it, never product code: a harness outside the repository is lost with its round.
- While building or after a review, a bug, an edge case, a name or a verified finding is decided and fixed without asking, a fix with its failing test first, and said in the summary; only what changes the agreed design or user-facing behavior is asked.
- At most two rounds per branch: after the second round's fixes, the tests, the end-to-end suite and a hand test cover the rest.

### Servers and names

- Never touch a real server without asking, even read-only; test on throwaway machines and delete them afterwards: a read can still load a server, and only its owner knows what runs there.
- Names in examples, tests, docs and commit messages are generic (for example `app`, `shop`, `vm1`, `example.com`), never a real project, client, server, domain or person, nor anything from an agent's memory or another project: a public history keeps whatever a commit once said.

### Docs

- The root holds what every task in the repository needs; a part only some tasks need is an area, `internals/<area>.md`, with one line in the root's index saying what it covers and when to read it: the root loads with every session, the areas only when their line says so.
- A root is kept under about 250 lines, a guide, never a cap: past it, what only some tasks need moves to an area, and no needed rule is cut or squeezed to fit.
- A rule's history goes in `internals/<area>.history.md`, or `internals/<section>.history.md` for a rule in a section of the root, starting with a line saying it is history, not rules, and its area's file wins where they differ.
- A history file is read only before changing or questioning one of its rules, when a review finding looks like it contradicts a decision, or when asked why.
- How a script or a function works, step by step, goes in its comments, never in an AGENTS.md or area: a comment changes with its code.
- A commit's reason goes in its body; only a decision a later session must respect becomes a rule, in its area or the root, its story in the history, in the same commit: anywhere else, the next session rediscovers it or contradicts it.
- Before adding a rule, find any line it repeats or contradicts, in its file and the root, and change that line instead: two lines on one thing drift apart.
- A line written or changed in an AGENTS.md or area is checked before its commit (for example `Push and tag only when asked: a push publishes, a tag releases.`):
  - one rule, imperative, no bold, no prose around it;
  - a why only when something real breaks without it, saying what breaks, never an invented one, which an agent takes for a reason;
  - examples written "(for example ...)" before the colon, and one colon at most outside code;
  - one reading only, else reworded, never explained further;
  - no story (review rounds, what broke, measurements), which goes in the history.
- A README says at a glance what the repository is and how to start; anything longer is a doc of its own in `docs/` that it links to: a README that piles up docs gets skimmed past.
- READMEs and docs are written plainly, with no selling words, no bold lead-in on every bullet, no closing summary, no emoji and no "Note" asides: they read as generated and bury the facts.
- Each fact has one home, and other docs link to it rather than repeat it: two copies drift apart.
- A README or doc rewritten (for example reworded, split or moved) gets a fact review against the old text and a separate tone review, told nothing of the changes and reading only for prose that sounds generated: a rewording changes facts its author misses.

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
