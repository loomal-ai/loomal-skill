---
name: loomal-skill
description: Loomal capabilities — agent inbox at mailgent.dev, encrypted credential vault with 2FA, calendar, and USDC payments. All actions are user-directed and scope-gated by a Loomal API key.
version: 0.1.1
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
        description: Loomal identity key (loid-...) from console.loomal.ai. The user grants the key its scopes; only those scopes' tools are available.
      - name: LOOMAL_API_URL
        required: false
        description: Override the API base URL. Defaults to https://api.loomal.ai. Used for staging environments.
    emoji: "📬"
    homepage: https://loomal.ai
---

# Loomal

Loomal gives an agent its own infrastructure: an inbox at `mailgent.dev`, an encrypted vault, a calendar, and USDC payments on Base. This skill helps the agent use those capabilities **when the user asks for them**.

## How it works

The user obtains a Loomal API key at [console.loomal.ai](https://console.loomal.ai), choosing which scopes to grant (mail, vault, calendar, payments, etc.). Only the granted scopes appear as tools — the API enforces this. If a tool the user asks about isn't available, the answer is "your Loomal key isn't scoped for that; visit console.loomal.ai to add the scope."

## One-time setup

Register the Loomal MCP server (the user runs this; it stores their API key locally):

```bash
openclaw mcp set loomal '{"command":"npx","args":["-y","@loomal/mcp"],"env":{"LOOMAL_API_KEY":"loid-..."}}'
```

Verify by asking the agent: *"who am I in Loomal?"* — the agent will call `identity_whoami` and echo the user's agent email and scopes back.

## Capabilities

The Loomal MCP server exposes tools across five namespaces. Use the right one when the user asks:

- **`identity_*`** — `whoami` (look up email, DID, scopes), `sign` and `verify` (sign data with the user's Ed25519 DID key when they ask for a signature).
- **`mail_*`** — read and send email from the user's `agent-xxx@mailgent.dev` address. Use `mail_send` for new threads, `mail_reply` to keep threading correct, `mail_list_messages` with `labels: ["unread"]` to find new mail.
- **`vault_*`** — read and write the user's encrypted credential store. `vault_store` saves a credential the user provides; `vault_get` retrieves one when the user asks the agent to use it. `vault_totp` returns the live 6-digit 2FA code (the seed itself stays encrypted and never leaves the vault).
- **`calendar_*`** — read and modify the user's calendar; `set_public` toggles a read-only iCal URL that the user can share.
- **`payments_*`** — `challenge` and `redeem` for x402 USDC settlement on Base when the user is selling a paid API endpoint, or when the user has explicitly asked the agent to pay for a service.

## Examples

These map to common user requests. The agent should always confirm with the user before taking actions that have side effects (sending mail, storing secrets, settling payments).

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

**"Pay $0.05 to the /search API at example.com"** (user-initiated)
- The user has explicitly asked to pay. Confirm the amount and recipient.
- Use the documented x402 flow: receive the 402 challenge, sign the `X-Payment` header with the user's wallet, retry.
- Show the user the signed receipt afterward.

## Working with secrets

Vault values (passwords, API keys, TOTP codes) are user-owned credentials. When the user asks the agent to *use* a credential — log in somewhere, sign a request, paste a 2FA code into a form — pass the value to the relevant tool or form field. The user can ask to see a credential at any time (`vault_get`). The default is to use values inline rather than echoing them into chat history, to keep transcripts clean — but this is a UX preference, not a secrecy rule. Always tell the user what the agent is doing with their credentials.

## Scopes are the trust boundary

If a tool isn't available, the user's API key isn't scoped for it. Tell the user which scope they need (`mail:send`, `vault:write`, `payments:accept`, etc.) and link them to [console.loomal.ai](https://console.loomal.ai) to add it. Don't try alternate tools to work around a missing scope — the user explicitly didn't grant it.

## Multiple Loomal identities

A user may have multiple Loomal identities (e.g., `loomal-sales` and `loomal-support`). Each identity is registered as a separate MCP server. Confirm with `identity_whoami` which one is active before sending mail or writing to the vault, especially if the user has more than one configured.

## Troubleshooting

- **Tool not found** → the user's key is missing the required scope. Direct them to console.loomal.ai.
- **401 / invalid key** → the user's key may be revoked or expired. Direct them to rotate at console.loomal.ai.
- **Email not arriving** → SES inbound takes a few seconds; suggest retrying `mail_list_messages` after 5 seconds. Check the spam label.
- **TOTP code rejected** → most often a clock-skew issue on the user's machine. RFC 6238 uses a 30-second window.

## Links

- Console (where the user gets their API key): <https://console.loomal.ai>
- Docs: <https://docs.loomal.ai>
- API reference: <https://docs.loomal.ai/api-reference>
- MCP package: <https://www.npmjs.com/package/@loomal/mcp>
