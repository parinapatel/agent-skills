---
name: integrate-flagsmith
description: >-
  Adds Flagsmith feature flags to any app. Default is the client-side
  environment key. Flag values only select an allowlisted mode or existing
  env var; secrets stay in Key Vault. Includes the shared LOCAL, DEV, staging,
  and production client-side keys and the test rules. Use when adding
  Flagsmith, feature flags, environment keys, flag rollout, or flag-driven
  behavior in any repository.
---

# Integrate Flagsmith

Apply this to any service or UI. Do not copy one app's flag names, branches, or error format into another app.

- UI: `https://flagsmith.dev.cloud.tdis.io/`
- API: `https://flagsmith.dev.cloud.tdis.io/api/v1/`

Override the host only when the app is explicitly pointed at a different Flagsmith.

## Keys

Use the **client-side environment key** in both backends and frontends. These keys are public SDK keys: they can read flags, and they cannot change flags or download the server-side environment document.

Application code reads **one** key from the environment for the deploy it is running. An empty key means the code default. Do not hardcode the table below into source. Copy the matching key into that environment's App Service setting or local `.env`.

A local or dev process gets the LOCAL or DEV key only. Do not point a non-prod process at the production key.

Server-side keys stay unused until a later review. Revisit one only when a flag must be invisible to browsers, or the API must evaluate from a downloaded environment document. Never write a server-side key into this skill, a repo, a frontend, or a log.

A JSON field does not hide a flag. Anyone with the client-side key can read every flag that is not marked server-side only.

## Shared environments

Give an app only the environments it deploys. Do not invent a missing stage. `test` and `ci` have no key and do not call Flagsmith.

| App label | Flagsmith environment | Client-side key |
|---|---|---|
| `local`, `development` | `LOCAL` | `6ZpxmFiTS4ZSkRjgkxnmMg` |
| `dev` | `DEV` | `MVKLqXF2dbJ7Lwarsasmxt` |
| `staging` | `staging` | `kJ3ZYVLfP2Ned67cVKao7j` |
| `production`, `prod` | `production` | `YF6EFUckgGcsCE3MBNufqq` |
| `test`, `ci` | none | empty |

The running process does not contain a map of every key. It uses the single key it was given.

Header for a manual read: `X-Environment-Key: <client-side key>` against `GET /api/v1/flags/`.

Ship with the flag off. Enable `LOCAL`, then `DEV`, then `staging` if it exists, then `production`. Rollback is turning the flag off.

## Flag values

A flag is a selector: a boolean, an enum, or a name. Secrets (passwords, connection strings, API keys, tokens, certificates) live in Key Vault and arrive as environment variables. The flag chooses which **already injected** variable to read, or which **already written** code path to run.

Resolve selectors through an allowlist in code. Unknown names, types, or extra fields use the code default.

Treat the string as data. Do not pass it to `eval`, a shell, SQL, a file path, a redirect, an auth issuer, or a URL host. Do not build a secret by concatenating a flag value with other text.

## Security rules

- The server fetches and evaluates its own flags. Ignore any flag value sent by the browser, a header, or a query param.
- Do not use a flag to grant access, skip authentication, choose an admin role, or weaken authorization. Those decisions stay in the auth stack.
- Code default is the current safe behavior. Missing key, Flagsmith down, unknown flag, or invalid JSON keeps that default and the process keeps serving.
- Log the flag name and the allowlisted mode only. Do not log the environment key or a raw flag payload.
- Restrict who can edit production flags in Flagsmith. A production flag change is a release action.
- Fetch once at startup. Later refreshes run in the background, so a request never waits on Flagsmith. A failed refresh keeps the last good snapshot, then the code default. One request uses one snapshot.

## Testing

Unit tests and CI never call Flagsmith and never use the keys in the table above.

Stub the SDK or the pure helper. Cover all of these:

1. Empty key: helper returns the code default and creates no client.
2. Flag missing: code default.
3. Flag off: current behavior stays.
4. Flag on with an allowlisted value: the new behavior runs.
5. Malformed JSON, wrong types, unknown enum, extra fields: code default.
6. A value that looks like a secret, URL, command, or path: rejected, code default.
7. SDK or network error: code default. If the cache layer is under test, a failed refresh keeps the last good snapshot.
8. API routes ignore a flag echoed by the client (header, query, or body).

Frontend tests mock the Flagsmith client the same way. Assert the UI does not send the flag to the API as something the API should trust.

A manual `GET /flags/` is for debugging only. Do not put a live call in CI.

## Writing a feature flag

Use this for every new flag, in any app. One flag is one decision.

- Name it `{service}_{behavior}` in snake_case. The name stays stable. Do not put a ticket number, a date, or a version in the name.
- The Flagsmith on/off switch is the master, and it ships off. The code default matches off, which is the behavior the app has today.
- Put the flag, still off, in every Flagsmith environment that app deploys before the code ships. Enable LOCAL, then DEV, then staging if it exists, then production.
- Optional JSON is a small set of selectors. Every field has a type and a default. Document the switch, each field, and the rollback in the Flagsmith flag description.
- A new field is optional. Old code ignores unknown fields only at the wire; the parser rejects unknown fields and wrong types and uses the code default. Do not reuse a field for a new meaning.
- Prefer one environment-level value. Add a Flagsmith identity only when the decision truly differs per caller. Do not send tokens, passwords, or raw secrets as identity traits.
- Do not use a flag for a secret, a permission, or config that should be a normal environment variable.
- After the behavior is permanent, delete the flag from code and from Flagsmith. Leave a short comment on the removal commit that names the flag.

Parser for that flag:

1. Switch off, missing flag, or empty key → code default.
2. Switch on and value null or empty → switch on, every selector at its default.
3. Switch on and JSON with only allowlisted fields of the right types → those values.
4. Anything else → code default.

Add the Testing cases for the new flag before calling the work done.

## Implementation sketch

1. Add the official Flagsmith SDK for that stack. Do not reimplement the client with `httpx` or `requests`. The SDK owns the HTTP call. Application code does not import `requests`.
2. Read the client-side key from the environment. Empty key means the code default.
3. Parse each flag into a typed value with an allowlist. Keep a pure function the rest of the app calls.
4. Fetch once at startup, then refresh in the background. On refresh failure, keep the last good snapshot, then the code default. A request reads the snapshot and does not call Flagsmith.
5. Add the tests in the Testing section before calling the work done.
