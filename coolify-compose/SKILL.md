---
name: coolify-compose
description: Make a Docker Compose file deploy correctly on Coolify — use Coolify's magic environment variables (SERVICE_URL_*, SERVICE_FQDN_*, SERVICE_PASSWORD_*, SERVICE_REALBASE64_*, SERVICE_HEX_*, SERVICE_USER_*), pick the right generated-value format for a secret's expected encoding, wire up healthchecks, domains, persistent volumes and networks, and debug compose resources that deploy but crash, hang, or lose their database credentials. Use this whenever the user mentions Coolify at all in connection with Docker Compose, docker-compose.yml/yaml, a "Docker Compose Empty" resource, a service template, or templates/compose/*.yaml — even if they only say things like "deploy this on Coolify", "make this work with Coolify", or paste a compose file and ask whether Coolify can run it.
---

# Making Docker Compose Coolify-compatible

Coolify does not run your compose file as-is. It parses it, rewrites parts of it, injects an `.env` it generates, injects its own network, and drives everything through its own proxy. Most "works locally, broken on Coolify" failures come from four places:

1. A secret the app needs that nobody set — because Coolify would have generated it, but only if you used the right magic variable name for the *encoding* the app expects.
2. Assumptions Coolify breaks: host-published ports, egress from `internal` networks, `container_name`.
3. Health checks that reference tools the image doesn't have, or a container that exits before the check can pass.
4. Rotating a generated value that is already baked into persistent state (databases, encrypted data).

Work through those in order. Most compose files need changes in only the first two.

## Workflow

### 1. Read the app's requirements, not just the compose file

Before touching YAML, establish what each secret must look like. An app that calls `base64_decode()` on a key and demands exactly 32 bytes cannot be given a random 32-character string — the decoded length will be wrong and it will fail at boot, usually with an unhelpful error. Grep the application for the env var names to find:

- the expected encoding (raw bytes, base64, hex, URL-safe base64)
- the expected decoded length
- whether the app *refuses to boot* without it or degrades gracefully
- whether the value is derivable or must come from a keygen CLI

An app that hard-fails on a missing key is the case to watch. Plainspace is a good example: it refuses to start in production unless all three `VAPID_*` values are present, so those must be `${VAR:?required}` in compose rather than optional. Check the app's entrypoint/Dockerfile for this too — a `HEALTHCHECK` in the Dockerfile is also the strongest signal of what the app considers its health endpoint.

### 2. Swap hardcoded secrets for Coolify magic variables

Coolify scans the compose for variables matching `SERVICE_*` and auto-creates them with generated values. Replace hardcoded passwords, and pick the generator whose output encoding matches what the app expects:

| Need | Use | Produces |
| --- | --- | --- |
| DB / admin password | `${SERVICE_PASSWORD_<NAME>}` | random password, no symbols |
| Long token / secret string | `${SERVICE_PASSWORD_64_<NAME>}` | 64-char random password, no symbols |
| Password that must include symbols | `${SERVICE_PASSWORDWITHSYMBOLS_<NAME>}` | random password with symbols |
| AES / HMAC key as base64 | `${SERVICE_REALBASE64_32_<NAME>}` | `base64_encode(random_bytes(32))` |
| 64-byte base64 key | `${SERVICE_REALBASE64_64_<NAME>}` | `base64_encode(random_bytes(64))` |
| Hex-encoded key | `${SERVICE_HEX_64_<NAME>}` | 64 hex chars (32 bytes) |
| Username | `${SERVICE_USER_<NAME>}` | 16-char random string |
| URL the app must know about | `${SERVICE_URL_<SERVICE>_<PORT>}` | the FQDN you assigned in the UI |

Two families look alike and are not interchangeable: `SERVICE_BASE64_32_*` is **not** base64 — it is a plain 32-character random string, despite the name. Only the `REALBASE64` variants encode bytes. Getting this backwards is the single most common magic-variable mistake, and it fails at runtime, not at parse time.

Reference each generated value consistently across services — define the variable once and interpolate it, rather than generating two independent values that must agree.

### 3. Mark what Coolify cannot generate as required

Magic variables cover passwords, keys, tokens, usernames. They do not cover asymmetric keypairs (VAPID, SSH, TLS), SMTP credentials, or API tokens from a third-party service. Those must come from the user, and should fail loudly rather than silently defaulting to empty:

```yaml
environment:
  - VAPID_PUBLIC_KEY=${VAPID_PUBLIC_KEY:?generate with: npx web-push generate-vapid-keys}
  - SMTP_HOST=${SMTP_HOST:?}
```

Compose's `:?` makes `docker compose up` abort with the message before any container starts, which turns a confusing runtime crash-loop into an actionable error in the deployment log. Use `:?` for anything the app cannot start without, `:-default` only for genuine optional config. Note the syntax difference — `:?` takes no default value, `:-` does.

### 4. Check the assumptions Coolify breaks

- **Ports.** If you assign a domain/FQDN to the service in the Coolify UI, its proxy routes traffic and `ports:` is unnecessary. Leaving `ports:` in place binds a host port that can collide with other stacks on the same machine — remove it unless the user specifically needs host-level access.
- **Egress.** A network marked `internal: true` has no route to the internet. This is a common hardening choice for databases and it will silently break anything on that network that sends mail, calls webhooks, or fetches packages. Whatever needs outbound internet must also be attached to a non-internal network.
- **`container_name`.** Coolify manages container names and labels itself. Drop `container_name` and let Coolify name things; hardcoding it fights the platform.
- **`version:`.** Obsolete in Compose v2 and ignored. Remove it.
- **Service names with hyphens/dots.** They work, but every magic variable derived from them normalizes hyphens and dots to underscores. `SERVICE_NAME_MY_APP` maps back to service `my-app`. Avoid surprises by keeping names alphanumeric with underscores where you can.
- **Image tags.** Pin to a specific version or digest. A floating tag makes deployments non-reproducible, and if the upstream image is built and published from `main`, `:latest` can change behavior under you between deploys.

### 5. Make the healthcheck work in the image you actually have

Coolify and Docker both run the healthcheck *inside* the container, using the tools present in that image. `curl -f http://localhost:3000/health` fails in a distroless or Alpine-without-curl image even though the app is perfectly healthy. Check the image:

- Alpine / Debian node or python images: `wget -qO-` usually works, `curl` often does not
- Distroless: neither works — remove the healthcheck rather than shipping one that always fails

Also give slow starters room, or a first-boot app will be marked unhealthy before it finishes booting:

```yaml
healthcheck:
  test: ["CMD", "wget", "-qO-", "http://127.0.0.1:3000/health"]
  interval: 30s
  timeout: 5s
  retries: 3
  start_period: 20s
```

Use `127.0.0.1`, not the service name, for an in-container check. And make dependent services wait on real readiness rather than hoping: `depends_on: { db: { condition: service_healthy } }`.

### 6. Verify before deploying, and know where the files land

```sh
docker compose config --quiet          # syntax + interpolation, fails on missing :? vars
docker compose up -d --wait            # --wait blocks until healthy or timeout
curl -fsS http://127.0.0.1:<port>/health
docker compose down -v                 # remove volumes too when testing from scratch
```

To test the real thing rather than a guess, export the magic variables yourself and run plain Compose locally — that proves the compose file is self-sufficient once Coolify supplies values. Coolify then writes the parsed file plus its generated `.env` to its services directory on the host, which is the first place to look when a deployed compose behaves differently from your local run.

## Debugging a compose resource that fails on Coolify

Read the container logs and match the symptom:

| Symptom | Cause | Fix |
| --- | --- | --- |
| Crash loop, `password authentication failed` / `28P01` | Generated DB password changed after the volume was initialized | Pin `SERVICE_PASSWORD_<NAME>` to the original value, or delete the volume and redeploy to re-initialize |
| Container exits immediately, "must be set in production" | Required app config has no value and no `:?` to catch it | Add `${VAR:?hint}` so compose fails before starting |
| `no such host` / DNS failure between services | Services on different networks, or `internal: true` blocking resolution of external names | Attach the calling service to the target's network |
| Health status stays unhealthy, app serves fine | Healthcheck tool missing in the image, or too-short `start_period` | Use a tool the image has; add `start_period` |
| Service starts but no domain works | No FQDN assigned, or the app needs `SERVICE_URL_*` for self-generated links | Assign the domain in the UI; reference `SERVICE_URL_<SERVICE>_<PORT>` |
| Silent data loss on redeploy | Named volume was declared but not actually mounted, or a `down -v` ran in a test | Verify `docker inspect` shows the mount |

The Postgres one deserves emphasis because it is silent and confusing: Coolify generates `SERVICE_PASSWORD_*` once and stores it, so it normally survives redeploys. The breakage comes from regenerating or editing it later. Any generated value that lands inside persistent state — a database volume, an encryption key protecting existing rows — is effectively write-once.

## One-click service templates

If the goal is to publish a template rather than just run something, the requirements are stricter: metadata comments at the top (`# port:` is mandatory, plus `# logo:`, `# slogan:`, `# category:`, `# tags:`, `# documentation:`), an SVG logo in the Coolify repo, no floating tags, and a paired docs page in a separate PR. The guide is <https://coolify.io/docs/contribute/service>; the rules for the required/optional variable syntax are there too.

## Reference

`references/magic-variables.md` — the complete generator list with exact algorithms, the underscore-counting rules that decide how Coolify parses a variable name, how `SERVICE_URL` / `SERVICE_FQDN` / `SERVICE_NAME` differ, and the parsing gotchas where a variable looks magic but is not generated. Read it when choosing a generator for an unusual secret, debugging why a `SERVICE_*` variable came out empty, or writing a template.
