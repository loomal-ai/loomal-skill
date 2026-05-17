---
name: loomal-skill
description: Loomal capabilities for BUYER projects — agent inbox at mailgent.dev, encrypted credential vault with 2FA, calendar, DID identity, and paying x402-priced URLs in USDC on Base under mandate caps. Not for SELLER projects (sellers use @loomal/sdk/paywall/* middleware in their server instead).
version: 0.2.0
metadata:
  openclaw:
    requires:
      env:
        - LOOMAL_API_KEY
      anyBins:
        - npx
        - bunx
    primaryEnv: LOOMAL_API_KEY
    envVars:
      - name: LOOMAL_API_KEY
        required: true
        description: Loomal identity key (loid-...) from console.loomal.ai. Must be from a BUYER project — SELLER keys won't surface most of these tools.
      - name: LOOMAL_API_URL
        required: false
        description: Override the API base URL. Defaults to https://api.loomal.ai. Used for staging environments.
    emoji: "📬"
    homepage: https://loomal.ai
---

# Loomal

Loomal gives a BUYER agent its own infrastructure: an inbox at `mailgent.dev`, an encrypted vault, a calendar, and the ability to pay x402-priced URLs in USDC on Base under per-call + daily caps. This skill helps the agent use those capabilities **when the user asks for them**.

## BUYER vs SELLER — and why this skill is BUYER-only

Loomal projects are either **BUYER** (agent does work, pays for tools, manages an inbox/vault/calendar) or **SELLER** (operates a paid x402 endpoint). This skill is for BUYER projects.

If the user's key is from a SELLER project, the MCP server fetches `/v0/whoami` at startup and surfaces almost nothing — SELLERs run a paid endpoint by importing `@loomal/sdk/paywall/express` (or hono / fastapi / mcp) into their own server code, not by driving tools from the agent. Direct them to [accept-payments docs](https://docs.loomal.ai/docs/for-seller/accept-payments) and don't try to work around the missing tools.

## How it works

The user obtains a Loomal API key at [console.loomal.ai](https://console.loomal.ai) when creating a BUYER project. The default BUYER scope set (`payments:spend`, `mail:read`, `mail:send`, `mail:manage`, `vault:read`, `vault:write`, `identity:sign`, `identity:verify`, `calendar:read`, `calendar:write`) makes the full buyer tool surface available. The user can narrow scopes per key for least-privilege.

## One-time setup

Register the Loomal MCP server (the user runs this; it stores their API key locally). The npm package version is pinned for supply-chain safety — bump it intentionally when you want a newer release:

```bash
openclaw mcp set loomal '{"command":"npx","args":["-y","@loomal/mcp@0.6.1"],"env":{"LOOMAL_API_KEY":"loid-..."}}'
```

Latest published version: <https://www.npmjs.com/package/@loomal/mcp>. Compare the published checksum against the source at <https://github.com/loomal-ai/loomal-mcp> if you want stronger provenance.

Verify by asking the agent: *"who am I in Loomal?"* — the agent will call `identity_whoami` and echo back the user's agent email, scopes, and `purpose` (should read `BUYER`).

## Recommended key hygiene

Loomal API keys are the delegated authority for the user's mail, vault, calendar, and payments. Recommend the user:

- Issue a separate key per task or per agent rather than a single broad key.
- Grant only the scopes the task needs (e.g., `mail:read` only, no `mail:send`, when the agent only needs to summarize email).
- Set a mandate with caps (`payments_mandates_create` with `maxPerCallUsdc` + `dailyCapUsdc`) so a runaway loop can't drain the wallet.
- Rotate keys when a task ends or when a key is no longer needed.
- Confirm `identity_whoami` matches the expected identity before any sensitive action, especially when multiple Loomal identities are configured.

## Capabilities

The Loomal MCP server exposes tools across five namespaces. Use the right one when the user asks:

- **`identity_*`** — `whoami` (look up email, DID, scopes, purpose), `sign` and `verify` (sign data with the user's Ed25519 DID key when they ask for a signature).
- **`mail_*`** — read and send email from the user's `agent-xxx@mailgent.dev` address. Use `mail_send` for new threads, `mail_reply` to keep threading correct, `mail_list_messages` with `labels: ["unread"]` to find new mail.
- **`vault_*`** — read and write the user's encrypted credential store. `vault_store` saves a credential the user provides; `vault_get` retrieves one when the user asks the agent to use it. `vault_totp` returns the live 6-digit 2FA code (the seed itself stays encrypted and never leaves the vault).
- **`calendar_*`** — read and modify the user's calendar; `set_public` toggles a read-only iCal URL that the user can share.
- **`payments_*`** — pay any x402-priced URL.
  - `payments_mandates_create` once to set per-call + daily USDC caps; reuse forever.
  - `payments_pay` with any x402-priced URL — Loomal runs the handshake server-side and returns the seller's content plus an on-chain `txHash`.
  - `payments_activity` for the bank-statement-style spend (and inbound, if relevant) history.
  - `payments_mandates_list` / `get` / `revoke` to manage the mandate.

## Confirmations required

Always ask the user to confirm before calling any of these tools — even if the user's request implied them, restate what's about to happen and wait for an explicit yes:

- `mail_send`, `mail_reply` — confirm the recipient, subject, and a short summary of the body.
- `mail_delete_message`, `mail_delete_thread`, `mail_update_labels` (when removing or archiving) — confirm which messages.
- `vault_store` — confirm the credential name and category (login, API key, etc.). Don't echo the secret value back.
- `vault_delete` — confirm by exact credential name.
- `vault_get`, `vault_totp` — when the user asks the agent to use a credential to log into a service, you don't need a separate confirmation for the lookup, but do tell the user which credential is being used.
- `calendar_create`, `calendar_update`, `calendar_delete` — confirm the event title, time, and attendees.
- `calendar_set_public` (toggling public visibility) — explicitly call out that an iCal URL will become readable by anyone with the link.
- `payments_pay` — real USDC settlement on Base. Confirm the URL and price (if known) before calling. Use `dryRun: true` first if you want to preview the mandate impact without spending.
- `payments_mandates_create` — confirm the per-call and daily caps before creating. First call installs an on-chain session key and can take 10–30s.
- `payments_mandates_revoke` — confirm by mandate id. After revocation, future `payments_pay` calls fail until a new mandate is created.

Read-only calls (`identity_whoami`, `mail_list_messages`, `mail_get_message`, `mail_get_thread`, `mail_get_attachment`, `vault_list`, `calendar_list`, `calendar_get`, `payments_activity`, `payments_mandates_list`, `payments_mandates_get`) don't require confirmation — they're safe to use to gather context for the user's request.

## Examples

These map to common user requests.

**"Check my inbox for unread emails"**
- `mail_list_messages` with `labels: ["unread"]`
- `mail_get_thread` for context if the user wants details
- `mail_reply` (not `mail_send`) when replying — preserves threading

**"Save this Stripe API key in my vault"**
- The user provides the key.
- `vault_store` with `type: "API_KEY"`, `metadata.service: "stripe"`.
- Confirm the credential name back to the user. Do not echo the key value.

**"Get my AWS 2FA code"**
- `vault_totp` against the credential the user named.
- Return the 6-digit code with its remaining validity window.

**"Schedule a meeting with Sarah for Tuesday at 2pm"**
- `calendar_create` with the user's timezone and Sarah's email as an attendee.
- Confirm the event details before saving if the user gave only partial info.

**"Pay $0.05 to the /search API at example.com"**
- The user has explicitly asked to pay. Confirm the URL and price.
- If no active mandate exists (call `payments_mandates_list` first), ask the user for caps and run `payments_mandates_create`.
- Run `payments_pay` with the URL. Loomal handles the x402 handshake and EIP-3009 signing under the project's wallet.
- Show the user the on-chain `txHash` and the content from the response.

**"How much have I spent on paid APIs this week?"**
- `payments_activity` with `limit: 50` (or higher).
- Sum the `direction: "out"` rows.

## Working with secrets

Vault values (passwords, API keys, TOTP codes) are user-owned credentials. When the user asks the agent to *use* a credential — log in to a service, sign a request, paste a 2FA code into a form — pass the value directly to the relevant tool or form field. **Do not echo secret values into chat output** unless the user explicitly asks to see them with a phrase like "show me the API key." Tell the user which credential name is being used and the destination service before using it.

If the agent is unsure which credential the user means (e.g., the user has multiple GitHub credentials in the vault), ask before reading.

## Scopes are the trust boundary

If a tool isn't available, the user's API key isn't scoped for it. Tell the user which scope they need (`mail:send`, `vault:write`, `payments:spend`, etc.) and link them to [console.loomal.ai](https://console.loomal.ai) to add it. Don't try alternate tools to work around a missing scope — the user explicitly didn't grant it.

If the user's key turns out to be from a SELLER project (only `payments:accept`), say so explicitly and direct them to create a BUYER project for agent work. Don't try to fake the buyer flow.

## Multiple Loomal identities

A user may have multiple Loomal identities (e.g., `loomal-sales` and `loomal-support`). Each identity is registered as a separate MCP server. Confirm with `identity_whoami` which one is active before sending mail or writing to the vault, especially if the user has more than one configured.

## Troubleshooting

- **Tool not found** → the user's key is missing the required scope, or the key is from a SELLER project. Direct them to console.loomal.ai.
- **401 / invalid key** → the user's key may be revoked or expired. Direct them to rotate at console.loomal.ai.
- **`payments_pay` returns `mandate_not_found`** → no active mandate. Call `payments_mandates_create` with the user's chosen caps.
- **`payments_pay` returns `mandate_per_call_exceeded` or `mandate_daily_cap_exceeded`** → the seller price (or cumulative day) is over the user's cap. Ask whether to raise the cap (`payments_mandates_create` again) or skip.
- **`payments_pay` returns `balance_insufficient`** → the user's wallet needs more USDC; direct them to top up via console.loomal.ai.
- **Email not arriving** → SES inbound takes a few seconds; suggest retrying `mail_list_messages` after 5 seconds. Check the spam label.
- **TOTP code rejected** → most often a clock-skew issue on the user's machine. RFC 6238 uses a 30-second window.

## Links

- Console (where the user gets their API key): <https://console.loomal.ai>
- Docs: <https://docs.loomal.ai>
- API reference: <https://docs.loomal.ai/api-reference>
- MCP package: <https://www.npmjs.com/package/@loomal/mcp>
- Seller path (not this skill): <https://docs.loomal.ai/docs/for-seller/accept-payments>
