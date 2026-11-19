# opencode-gemini-router

Native **OpenCode V2** plugin ported from
[`pi-gemini-router`](https://github.com/hakergeniusz/pi-gemini-router). It adds
two router models to the `google` provider that walk a model ladder and rotate
API keys until one pair answers, with per-pair quota awareness:

```
google/gemini-flash        gemini-4-hydrogen → gemini-flash-latest → 3.8 → 3.7 → 3.6 → 3.5 → 3-flash-preview
google/gemini-flash-lite   gemini-flash-lite-latest → 3.5-flash-lite → 3.1-flash-lite
```

The router holds its `(model, key)` pair until that pair fails; success, titles,
and continuations do not move it. Missing pairs (a key whose project lacks a
model) and quota-dead pairs are skipped. A walk covers every pair once; a
successful response ends the walk and a new prompt starts a fresh one.

## Load

Add it to `opencode.json(c)` (global or project-local):

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "plugins": [
    {
      "package": "opencode-gemini-router",
      "options": {
        "order": "key-first",
        "apiKeys": ["${GEMINI_KEY_1}", "${GEMINI_KEY_2}"] // if your config substitutes env vars
      }
    }
  ]
}
```

A local checkout works too: `"plugins": ["./plugins/opencode-gemini-router"]`
or `/absolute/path/opencode-gemini-router`. Do not commit real API keys —
prefer the `GEMINI_ROUTER_API_KEYS` environment variable (comma separated),
which is used when `apiKeys` is not supplied.

## Options

All options come from the `plugins` object form (`ctx.options`). A JSON file at
`~/.config/opencode/gemini-router.json` is also read first, with `ctx.options`
taking precedence; override the path with `configPath`.

| Option | Default | Meaning |
|---|---|---|
| `ladders` | built-in ladders above | `{"gemini-flash": [...], "gemini-flash-lite": [...]}` |
| `order` | `"key-first"` | `key-first`: all keys on one model, then next model. `model-first`: whole ladder per key. |
| `quotaAware` | `true` | Remember 429s per `(model, key)` and skip dead pairs. |
| `apiKeys` | `GEMINI_ROUTER_API_KEYS` | Ordered API key list; with more than one, the `http.request` hook rotates them. |
| `keyHeader` | `"x-goog-api-key"` | Header used to send the rotated key. |
| `googleProviderID` | `"google"` | Provider the router models are attached to. |
| `configPath` | `~/.config/opencode/gemini-router.json` | Optional JSON config file. |

## Commands

```
/gemini-router              # show ladders, key count, advance order, quota marks
/gemini-router reset        # restart at the ladder head on key1
/gemini-router quota-reset  # clear all quota marks
```

Status is written into the session as a synthetic message (OpenCode plugins
have no `ui.notify` equivalent).

## Quota awareness

A 429 is classified from its body and remembered per `(model, key)` pair in
plugin `ctx.storage` (survives restarts):

- **RPD** (`PerDay` / "per day") → dead until the next midnight Pacific.
- **Per-minute** (RPM/TPM, often `Please retry in Xs`) → cooling, capped at 120s.
- **Ambiguous quota 429** → possibly RPD, skipped with a 30-minute probe TTL.
- A **successful response** clears the pair's mark. If every pair is marked,
  the router falls back to a held pair instead of throwing.

## How the pi extension maps to OpenCode V2

| pi concept | OpenCode V2 used here |
|---|---|
| `registerVirtualModel` + `route` | `ctx.provider.transform` registers the models; `session.hook("experimental.model.resolve")` routes each request |
| `google-keyN` key rotation | `session.hook("http.request")` rewrites the API-key header (see gaps) |
| `agent_before_settle` rotation | `session.hook("retry")` advances the pair and makes quota errors retryable |
| session custom entries for state | `ctx.storage` (`state/<sessionID>`), plus an in-memory per-session cache |
| `~/.pi/agent/gemini-router*.json` | `ctx.options` and `~/.config/opencode/gemini-router.json`; quota marks in `ctx.storage` |
| thought-signature injection / xhigh / versioned URL | `session.hook("http.request")` rewrites the native request body and URL |
| `/gemini-router` via `registerCommand` | `ctx.command.transform` + `ctx.session.synthetic` for output |
| `turn_end` / `before_agent_start` | `retry` hook + `http.response` hook (success clears marks) |

## Known gaps / differences

- **No `google-keyN` providers.** OpenCode's model-resolve hook cannot switch
  to a model from another provider, so pi-multi-account's per-account providers
  are not reproduced. Key rotation instead sends each key through the
  `x-goog-api-key` header of the single `google` provider (change with
  `keyHeader`). This preserves the quota-multiplying intent, not the exact
  provider topology.
- **Model rotation on retry is best-effort.** OpenCode caps the physical retry
  count, and it is not documented whether a retry re-runs the model-resolve
  hook. Pair state is advanced and the key is rotated on the wire regardless;
  a ladder deeper than OpenCode's retry budget may need a new prompt to finish.
- **`signal.ts` not ported.** The duck-typed abort-signal crash was a pi-ai
  adapter bug; OpenCode's native provider owns abort handling, so there is no
  equivalent patch.
- **Wire fixes are best-effort.** Thought-signature sentinel injection and the
  xhigh → v1alpha rewrite run on the native HTTP body and are applied for
  Gemini 3+ models only when `xhigh` was detected in the request options. If
  OpenCode's Google driver sends a different body shape, those two fixes become
  no-ops (routing/quota behavior is unaffected).
- Ladder entries must exist on the configured provider; unavailable models are
  skipped. The available-model snapshot is taken at plugin setup, so a catalog
  refresh during a session is picked up after reload.

## Typecheck & test

```bash
bun test/router.test.ts                              # pure rotation/quota logic
node --experimental-strip-types test/router.test.ts  # same, on Node 22+
tsc --noEmit                                         # after installing devDependencies
```

## License

MIT — 2026 hakergeniusz.
