History, not rules: AGENTS.md's Rules win where they differ.

- `latest` with one release kept was decided on 2026-10-29, over an exact pin bumped by hand.
- `deploy: {replicas: 2}` dropped both connectors together on a release or a token rotation, and Cloudflare answered 1033 until one reconnected. YAML anchors work within a file only.
- Without the healthcheck `--wait` only waited for running, and the gap is the connection time, during which a release went on to the second connector and the tunnel had none. A token nobody accepts leaves a connector retrying forever: it never exits on it (verified).
- A proxied CNAME from a zone of another account gives error 1033, as when no connector is up (verified).
- Every network a project owns is one more thing to name and check, hence none but `tunnel`.
- The services ran as the image's own 65532, stated in `user:` so an upstream change never moved it, until 2026-11-11: 65532 is every distroless image's `nonroot`, so a process escaped from any container on that uid reached the connectors' environment (the token) through `/proc`. They run as the environment's `DEVOPSY_UID` since, a static binary that writes nothing running as any uid (checked on a test machine with no passwd entry).
