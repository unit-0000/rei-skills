---
name: pi-reilabs-provider
description: Install the Rei Labs (reilabs) custom provider into the user's pi.dev config so that rei-qwen3-coder, rei-coder-lite, rei-coder-pro, and rei-coder-max become available. Use when the user asks to add, enable, or install Rei Labs models, the reilabs provider, or coder.reilabs.org in pi (pi.dev / @mariozechner/pi-coding-agent).
---

# Install the Rei Labs provider for pi.dev

Add a `reilabs` entry under `providers` in the user's pi `models.json`, then store the API key in `auth.json`, so the four Rei Labs coder models are selectable via `/model`. The provider uses the `openai-completions` API pointed at `https://coder.reilabs.org/v1`.

This skill supports Linux, macOS, and Windows (including WSL).

## Why `auth.json` and not just an env var

pi resolves a provider's API key in this order: `auth.json` → `models.json` `apiKey` field → undefined. If `models.json` has `apiKey: "REILABS_API_KEY"` and the env var is not set, pi's `resolveConfigValue` silently falls back to the literal string and sends `Authorization: Bearer REILABS_API_KEY` — **pi does not warn at startup or at request time; the server just returns 401.** Storing the real key in `~/.pi/agent/auth.json` avoids that silent-fail because auth.json is checked first. The file is created with `0600` permissions so the key is not world-readable.

Note: pi's validator **requires** `apiKey` (or `oauth`) in a `models.json` provider block that defines models — the block will fail to load without it. The `apiKey` field is therefore kept as a required placeholder, but `auth.json` takes priority at request time and is where the real key lives.

pi has no `/connect`-style interactive prompt for providers registered via `models.json`. `/login` only lists built-in providers and providers that register an `oauth` handler via a TypeScript extension.

## Canonical provider block

Merge this under `providers` in `~/.pi/agent/models.json`. The `apiKey` field is required by pi's validator; the actual key is stored in `auth.json` (see step 3) and overrides this value at request time:

```json
"reilabs": {
  "baseUrl": "https://coder.reilabs.org/v1",
  "api": "openai-completions",
  "apiKey": "REILABS_API_KEY",
  "headers": {
    "User-Agent": "pi/0.70.0"
  },
  "models": [
    { "id": "rei-qwen3-coder", "name": "Rei Qwen3 Coder" },
    { "id": "rei-coder-lite",  "name": "Rei Coder Lite" },
    { "id": "rei-coder-pro",   "name": "Rei Coder Pro" },
    { "id": "rei-coder-max",   "name": "Rei Coder Max" }
  ]
}
```

The `User-Agent` override is required. pi uses the official `openai` Node SDK which sends `User-Agent: OpenAI/JS <ver>`, and the `coder.reilabs.org` gateway's WAF blocks that UA with `403 Your request was blocked` before auth is even checked. Overriding the header to anything that doesn't match the OpenAI-SDK pattern (e.g. `pi/<ver>`) bypasses the block.

## Procedure

1. **Pick the target config files.**
   - Linux/macOS: `~/.pi/agent/models.json` and `~/.pi/agent/auth.json`
   - Windows (native): `%USERPROFILE%\.pi\agent\models.json` and `%USERPROFILE%\.pi\agent\auth.json`
   - Windows (WSL, recommended): the Unix paths above, inside the WSL distro
   - pi does not have a project-local override for these files; always edit the user-global ones.
   - If `models.json` does not exist, create it (and `~/.pi/agent/` if needed) with: `{ "providers": {} }`.
   - If `auth.json` does not exist, create it with `{}` and chmod it to `0600` (Linux/macOS). Do not create `auth.json` on Windows with chmod — rely on `%USERPROFILE%` ACLs.

2. **Install the provider block.**
   - Read and parse `models.json` as JSON. Keep the original text in memory for rollback.
   - Merge the canonical block as `providers.reilabs`. Preserve every other top-level key and every other provider under `providers`.
   - Idempotency:
     - If `providers.reilabs` is absent, add it.
     - If present and deep-equal to the canonical block, report "provider already installed" and skip the write.
     - If present but different, show the user a diff and ask before overwriting.
   - Write with 2-space indentation and a trailing newline. Re-parse to validate; on parse failure, restore the pre-write text and stop.

3. **Prompt the user for the API key and write it to `auth.json`.**
   - Ask the user for their Rei Labs API key interactively. Do not log it or echo it back in full; show only a short masked prefix like `sk-...1a2b` when confirming.
   - Read and parse `auth.json` (creating `{}` if missing). Keep original text in memory.
   - Set `auth["reilabs"] = { "type": "api_key", "key": <entered-key> }`.
   - Preserve every other top-level key (other provider credentials must remain untouched).
   - Idempotency: if `reilabs` already exists with a non-empty `key`, ask the user whether to replace it before overwriting.
   - Write with 2-space indentation. On Linux/macOS, re-apply `0600` permissions after writing. Re-parse to validate; on failure, restore and stop.
   - If the user declines to enter a key right now, leave `auth.json` alone and warn them clearly: the provider is installed but **no chat request will succeed until `auth.json` has a real key or `REILABS_API_KEY` is exported**. Without one of those, pi will send the literal string `"REILABS_API_KEY"` as the bearer token and the server will return 401 with no pi-side warning.

4. **Report.** Print the final list of provider keys and call out the four newly available models:
   - `reilabs/rei-qwen3-coder` — Rei Qwen3 Coder
   - `reilabs/rei-coder-lite` — Rei Coder Lite
   - `reilabs/rei-coder-pro` — Rei Coder Pro
   - `reilabs/rei-coder-max` — Rei Coder Max

   pi reloads `models.json` each time `/model` is opened, so no restart is required. If pi is already running, the user can open `/model` to see the new entries.

## Alternative key storage

If the user prefers not to keep the raw key in `auth.json`, the `key` field also accepts:

- **Shell command:** `"key": "!security find-generic-password -ws 'reilabs'"` (macOS keychain) or `"key": "!op read 'op://vault/reilabs/key'"` (1Password CLI). stdout is used as the key.
- **Environment variable name:** `"key": "REILABS_API_KEY"` resolves from the named env var at request time. Same silent-fail caveat as putting the env var name in `models.json` — if the var is unset, pi will not warn until a request is made.

Only suggest these if the user asks; `auth.json` with the literal key is the default.

## Out of scope

- Do not modify default model selection, agent bindings, keybindings, or other providers.
- Do not install npm packages; pi resolves the `openai-completions` streaming adapter internally.
- Do not write the raw API key into `models.json`. Keys belong in `auth.json` (or a shell-command/env-var reference). Keep the `apiKey` field in `models.json` as the literal string `"REILABS_API_KEY"` — it's required by the validator but will be overridden by `auth.json` at request time.
- Do not drop the `headers.User-Agent` override. Without it, every chat request returns `403` from the gateway WAF.
- Do not create backup files unless the user asks — the pre-write in-memory copies are sufficient for rollback within this run.
