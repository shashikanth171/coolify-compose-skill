# Coolify magic environment variables — reference

Verified against Coolify source (branch `next`, and a live 4.3.23 instance): `bootstrap/helpers/shared.php` (`generateEnvValue`, `parseEnvVariable`), `bootstrap/helpers/parsers.php` (where variables get created), `app/Jobs/ApplicationDeploymentJob.php` (where `SERVICE_URL` / `SERVICE_FQDN` / `SERVICE_NAME` are emitted), `app/Console/Commands/Generate/Services.php` (the template-time rewrite).

Coolify scans a compose resource for `SERVICE_*` variables and creates them as real environment variables with generated values. Understanding *which* strings qualify, and what each generates, is what separates a compose that works from one that fails at runtime.

## Contents

- [The generators](#the-generators)
- [Choosing by expected encoding](#choosing-by-expected-encoding)
- [How the name is parsed](#how-the-name-is-parsed)
- [Variables Coolify sets for you](#variables-coolify-sets-for-you)
- [Required vs optional syntax](#required-vs-optional-syntax)
- [Ordering and evaluation](#ordering-and-evaluation)
- [Cases where generation silently does nothing](#cases-where-generation-silently-does-nothing)
- [Rotating a generated value](#rotating-a-generated-value)

## The generators

`generateEnvValue($command)` switches on the middle segment. `<NAME>` is a free-form suffix that makes the variable unique within the resource — it has no meaning to Coolify, only to the compose file that references it.

| Variable | Generates |
| --- | --- |
| `SERVICE_PASSWORD_<NAME>` | `Str::password(symbols: false)` — random password, default length 32, no symbols |
| `SERVICE_PASSWORD_64_<NAME>` | `Str::password(length: 64, symbols: false)` |
| `SERVICE_PASSWORDWITHSYMBOLS_<NAME>` | `Str::password(symbols: true)` |
| `SERVICE_PASSWORDWITHSYMBOLS_64_<NAME>` | `Str::password(length: 64, symbols: true)` |
| `SERVICE_BASE64_<NAME>` / `SERVICE_BASE64_32_<NAME>` | `Str::random(32)` — a plain 32-char string. **Not base64**, despite the name |
| `SERVICE_BASE64_64_<NAME>` | `Str::random(64)` |
| `SERVICE_BASE64_128_<NAME>` | `Str::random(128)` |
| `SERVICE_REALBASE64_<NAME>` / `SERVICE_REALBASE64_32_<NAME>` | `base64_encode(random_bytes(32))` — 44 chars, decodes to exactly 32 bytes |
| `SERVICE_REALBASE64_64_<NAME>` | `base64_encode(random_bytes(64))` |
| `SERVICE_REALBASE64_128_<NAME>` | `base64_encode(random_bytes(128))` |
| `SERVICE_HEX_32_<NAME>` | `bin2hex(random_bytes(16))` — 32 hex chars |
| `SERVICE_HEX_64_<NAME>` | `bin2hex(random_bytes(32))` — 64 hex chars |
| `SERVICE_HEX_128_<NAME>` | `bin2hex(random_bytes(64))` — 128 hex chars |
| `SERVICE_USER_<NAME>` | `Str::random(16)` — mixed case |
| `SERVICE_LOWERCASEUSER_<NAME>` | `Str::lower(Str::random(16))` |
| `SERVICE_SUPABASEANON_<NAME>` | An anon-role JWT signed with `SERVICE_PASSWORD_JWT` |
| `SERVICE_SUPABASESERVICE_<NAME>` | A `service_role` JWT signed with `SERVICE_PASSWORD_JWT` |

The Supabase pair is the only generator that depends on another variable: `SERVICE_PASSWORD_JWT` must already exist, otherwise the function returns `null` and the JWT variable is created empty.

## Choosing by expected encoding

Match the app's expectation, not the name you remember:

| App expects | Use | Why |
| --- | --- | --- |
| Any opaque password, used in a connection string | `SERVICE_PASSWORD_<NAME>` | no `+`/`/`/`=` to break URLs or YAML |
| A secret that must be URL/path safe | `SERVICE_PASSWORD_<NAME>` | symbols excluded on purpose |
| `base64_decode($key)` yielding exactly 32 bytes (AES-256, HMAC) | `SERVICE_REALBASE64_32_<NAME>` | the only family whose output actually decodes to 32 bytes |
| `base64_decode($key)` yielding 64 bytes | `SERVICE_REALBASE64_64_<NAME>` | |
| A hex string of N chars | `SERVICE_HEX_<N>_<NAME>` | hex is half the length of the bytes it encodes |
| Just "a long random string" | `SERVICE_PASSWORD_64_<NAME>` | |

Two real traps:

- **`SERVICE_BASE64_*` is a misnomer.** It returns an unencoded random string. Feed it to an app that base64-decodes and you get garbage of the wrong length, which usually surfaces as a boot failure inside the crypto layer rather than a config error.
- **Symbol-bearing passwords break connection strings.** `SERVICE_PASSWORDWITHSYMBOLS_*` can emit `@ : / # ? % &`, which corrupt a `postgres://user:pass@host/db` URL. Reserve it for values that are never embedded in a URL.

## How the name is parsed

`parseEnvVariable()` counts underscores in the variable name and branches on the total. Only names with exactly 2 or 3 underscores are treated as magic:

- **2 underscores** — `SERVICE_USER_APP`, `SERVICE_REALBASE64_SECRET`, `SERVICE_URL_APP`:
  - if it starts with `SERVICE_FQDN` or `SERVICE_URL`, the command is the segment after `SERVICE_` and the last segment is the service name;
  - otherwise the command is everything between `SERVICE_` and the final segment.
- **3 underscores** — `SERVICE_URL_APP_3000`, `SERVICE_REALBASE64_32_APP`, `SERVICE_PASSWORD_64_ADMIN`:
  - if it starts with `SERVICE_FQDN` or `SERVICE_URL`, the segments are command / service / port, and the port is only accepted if it validates as an integer;
  - otherwise the command is everything between `SERVICE_` and the final segment, so the length prefix (`32`, `64`, `128`) is part of the command.

Consequences worth remembering: the suffix should stay a single underscore-free token, since a two-word suffix like `SERVICE_PASSWORD_MY_APP` parses as command `PASSWORD`, suffix `MY_APP` — which works, but only by luck of the last-segment rule. And anything with 4+ underscores is not magic at all, so it lands as an ordinary unset variable.

Hyphens and dots in *service* names are normalized when Coolify derives variable keys: `str($serviceName)->replace('-', '_')->replace('.', '_')->upper()`. Service `my-app` becomes `MY_APP` in `SERVICE_NAME_*` and `SERVICE_URL_*`.

## Variables Coolify sets for you

These are not generated from a magic command — they are computed at deploy time from the UI state and the compose file, and pushed into the `.env` next to the other values:

| Variable | Source | Notes |
| --- | --- | --- |
| `SERVICE_URL_<SERVICE>_<PORT>` | the FQDN assigned to that service, scheme included | absent until a domain is assigned in the UI |
| `SERVICE_FQDN_<SERVICE>` | the same FQDN, host only | |
| `SERVICE_NAME_<SERVICE>` | the service key from the compose file | lets one service reference another by name |

Because `SERVICE_URL_*` only exists once a domain is configured, an app that builds self-referential links (invitation emails, share URLs, OAuth callbacks) will get an empty string in a fresh resource until the user assigns a domain. Reference the variable anyway and document the dependency — hardcoding a placeholder URL is worse, because it silently sends links to the wrong place.

When Coolify generates the official template files, it rewrites `SERVICE_URL` to `SERVICE_FQDN` (`app/Console/Commands/Generate/Services.php`), so a compose borrowed from an official template may use the host-only form. Both exist in deployed resources.

## Required vs optional syntax

Plain Compose, not a Coolify feature, but the mechanism Coolify relies on to fail fast:

```yaml
- SECRET=${SECRET:?must be set}          # abort if unset or empty
- PORT=${PORT:-8080}                     # fall back to a default
- USER=${SERVICE_USER_APP:-app}          # generated, with a default
```

`:?` takes no default. Compose refuses to start when it is violated, so the deployment log carries the hint text instead of the container crash-looping. Use it for anything the app cannot boot without — including values Coolify cannot generate, such as asymmetric keypairs, SMTP credentials, and third-party API tokens.

## Ordering and evaluation

At deploy, Coolify sorts runtime variables so that any variable whose *value* references a `SERVICE_*` variable is emitted after the ones that don't. That is why a chain like

```yaml
- DATABASE_URL=postgres://${SERVICE_USER_POSTGRES}:${SERVICE_PASSWORD_POSTGRES}@db:5432/app
```

resolves reliably even though `DATABASE_URL` itself is not generated. Referencing generated values inside other variables is supported and idiomatic; generating the composed value separately is not needed.

Values reach containers two ways: the `.env` Coolify writes beside the compose file, and direct injection into the service's `environment:`. Compose interpolation handles the first, and is the reason `${VAR}` works inside a compose file without any shell involvement.

## Cases where generation silently does nothing

- **Unrecognized command segment.** The `default` branch returns `null`. A typo like `SERVICE_PASSWORDWIDTHSYMBOLS_X` or `SERVICE_REALBASE_32_X` creates an empty variable and deploys happily; the app then fails on an empty key. If a `SERVICE_*` variable shows up empty in the UI, suspect a misspelled command rather than a Coolify bug.
- **4+ underscores.** Falls outside the parser entirely and is treated as a normal variable you must set yourself.
- **Supabase JWTs without `SERVICE_PASSWORD_JWT`.** The dependent variable is created empty.
- **A `SERVICE_URL_*` with no domain assigned.** Not an error; the variable simply is not generated.
- **Values referenced but never declared.** A compose can interpolate `${SERVICE_PASSWORD_POSTGRES}` into a connection string without ever listing it under `environment:` — Coolify still generates it, but you will not see it listed in the service's own environment block. Check the resource's environment variables, not just the compose, when hunting down a value.

## Rotating a generated value

Generated values are stored, so they survive redeploys. That makes them effectively write-once whenever the value is baked into persistent state:

- **Database credentials.** Postgres (and MySQL, Mongo, …) initialize a volume with the password present at first start. Changing `SERVICE_PASSWORD_POSTGRES` afterwards yields `password authentication failed for user` / SQLSTATE `28P01` and a container that crash-loops while the database itself is healthy and holding data. Either pin the variable back to its original value or delete the volume so the database re-initializes with the new one.
- **Encryption keys.** An app that encrypts rows at rest will no longer be able to decrypt existing rows. Plainspace is exactly this case: its two email keys encrypt `members.email`, and losing or changing them makes stored addresses permanently unreadable. Escrow them off-host before the first deploy.

When a compose is first tested locally, `docker compose down -v` is the fastest way to guarantee a clean re-initialization — and the fastest way to lose data if you forget which environment you are pointed at.
