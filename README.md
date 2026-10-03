# coolify-compose

An agent skill that makes a Docker Compose file deploy correctly on [Coolify](https://coolify.io).

Coolify does not run your compose file as-is. It parses it, rewrites parts of it, injects an `.env` it generates, injects its own network, and routes traffic through its own proxy. Most "works locally, broken on Coolify" failures come from a short list of predictable causes — and almost all of them are fixable in the compose file without touching Coolify's UI.

## What it does

Teaches an agent to:

- **Pick the right magic variable** for each secret, based on the encoding the application actually expects (`SERVICE_PASSWORD_*`, `SERVICE_REALBASE64_*`, `SERVICE_HEX_*`, `SERVICE_USER_*`).
- **Mark un-generatable values as required** (`${VAR:?}`) so Compose fails before any container starts, instead of crash-looping on a missing key.
- **Fix the assumptions Coolify breaks** — host-published ports, `internal: true` blocking egress, `container_name`, floating image tags.
- **Write healthchecks that work in the image you have** — the check runs inside the container, so `curl` in a distroless image is a broken check, not a broken app.
- **Debug deployed-but-broken resources** with a symptom → cause table, including the Postgres `28P01 auth_failed` trap that comes from regenerating a stored database password.

A companion reference, [`references/magic-variables.md`](coolify-compose/references/magic-variables.md), documents every generator with its exact algorithm, the underscore-counting rules that decide how Coolify parses a variable name, and the cases where generation silently produces nothing.

## Install

Copy the skill directory into wherever your agent reads skills:

```sh
git clone https://github.com/shashikanth171/coolify-compose-skill.git
cp -r coolify-compose-skill/coolify-compose ~/.claude/skills/
```

Agents that read OpenCode's default skill location (`~/.config/opencode/skills/`) or per-project `.opencode/skills/` work the same way — drop the `coolify-compose/` directory in and it will be discovered on the next session.

Then just ask for the thing you want, for example:

> Make this docker-compose.yml work on Coolify — it needs Postgres and a generated signing key.

## Why it exists

The rules were derived by reading Coolify's own source rather than guessing at its behaviour, then confirmed against a live 4.3.23 instance — including the failures. Two details that are easy to get wrong and expensive to debug in production:

- `SERVICE_BASE64_32_*` is a **misnomer**: it returns an unencoded random string, not base64. Only the `SERVICE_REALBASE64_*` family encodes bytes. Handing the wrong one to an app that base64-decodes an AES key produces a key of the wrong length and a boot failure inside the crypto layer.
- An unrecognized command segment hits a `default: return null` branch, so a **typo produces an empty variable and a deploy that looks successful** until the application dies on an empty key.

Coolify is licensed under Apache-2.0. This skill documents observed behaviour and quotes only short functional expressions; no Coolify source code is included.

## License

GPL-3.0. See [LICENSE](LICENSE).

If you find a rule that has drifted — Coolify's compose handling changes between versions — please open an issue or a pull request. Corrections to the reference are as welcome as the original.
