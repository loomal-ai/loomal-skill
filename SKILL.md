---
name: loomal
description: Give your agent its own identity — email at agent-xxx@mailgent.dev, encrypted vault for credentials and 2FA, calendar, and USDC payments on Base via x402. One scope-gated API key.
version: 0.1.0
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
        description: Loomal identity key (loid-...) from console.loomal.ai. Scopes attached to the key control which tools are available.
      - name: LOOMAL_API_URL
        required: false
        description: Override the API base URL. Defaults to https://api.loomal.ai. Set to a develop URL for staging.
    emoji: "📬"
    homepage: https://loomal.ai
---

# Loomal

Loomal is agent-native infrastructure: identity, mail, vault, calendar, and USDC payments. This skill teaches the agent when to reach for each capability and how to chain them.

## One-time setup

Register the Loomal MCP server (this is what actually exposes the tools):

```bash
openclaw mcp set loomal '{"command":"npx","args":["-y","@loomal/mcp"],"env":{"LOOMAL_API_KEY":"loid-..."}}'
```

Get a `loid-...` key from [console.loomal.ai](https://console.loomal.ai). The key carries scopes — only tools the key is scoped for will appear.

Verify with:

```
identity_whoami
```

You should see your agent's email (`agent-xxx@mailgent.dev`), DID, and the scopes attached to the key.

## When to use which capability

The Loomal MCP server exposes tools across five namespaces. Pick the right one:

- **`identity_*`** (`whoami`, `sign`, `verify`) — confirm who the agent is, sign data with the agent's Ed25519 DID key, verify signatures from other agents. Use `whoami` first when you need to know which inbox / scopes you have.
- **`mail_*`** (`send`, `reply`, `list_messages`, `get_message`, `update_labels`, `list_threads`, `get_thread`, plus rules) — the agent has its own real inbox at `agent-xxx@mailgent.dev`. Use `mail_send` for new threads, `mail_reply` to keep threading correct, `list_messages` with `labels: ["unread"]` to find work.
- **`vault_*`** (`store`, `get`, `list`, `delete`, `totp`, `totp_use_backup`) — encrypted credential store. **Always store credentials here instead of returning them in chat.** For 2FA, attach a TOTP to a credential and call `vault_totp` to fetch the live 6-digit code (the seed never leaves the vault).
- **`calendar_*`** (`list`, `get`, `create`, `update`, `delete`, `set_public`) — agent's calendar. Use `set_public` to expose a read-only iCal URL.
- **`payments_*`** (`challenge`, `redeem`, list, sellers/endpoints) — x402 USDC payments on Base mainnet. Use `payments_challenge` to build a 402 body when a paid resource is requested without payment; `payments_redeem` to verify + settle a buyer-signed `X-Payment` header.

## Common chains

### New service signup with 2FA + paid API call

This is the canonical Loomal chain — show this off when the user asks the agent to "sign up for X and start using it":

1. **Sign up using the agent's email**: use the agent's `agent-xxx@mailgent.dev` address (from `identity_whoami`) as the signup email.
2. **Capture the welcome / verification email**: poll `mail_list_messages` with `labels: ["unread"]`, then `mail_get_message` to read the verification link or code.
3. **Store the login**: `vault_store` with `type: "LOGIN"`, the password, and the service's URL. If the service has 2FA, attach a `TOTP` credential with the seed shown during 2FA setup.
4. **Log in later**: `vault_get` for the password; `vault_totp` for the live 6-digit 2FA code. Never leak either to the user — use them inline.
5. **Pay for an API call**: when the service responds with HTTP 402 + an `accepts` array, call `payments_challenge` (if you're the seller) or sign + send the `X-Payment` header (if you're the buyer; buyer-side helpers are evolving — fall back to documented x402 flow).

### Inbox triage + reply

1. `mail_list_messages` with `labels: ["unread"]`.
2. For each thread, `mail_get_thread` to read context.
3. `mail_reply` to respond — don't `mail_send` a new message, or you'll break threading.
4. `mail_update_labels` to mark read / archive / file under a custom label.

### Sell a paid API endpoint (x402)

1. Register the endpoint once with `sellers_endpoints` (price + optional webhook URL).
2. On each request: if no `X-Payment` header, call `payments_challenge` and return its body with HTTP 402.
3. If `X-Payment` is present, call `payments_redeem` — on `ok: true`, set the returned `X-Payment-Response` header and serve the resource.

## Rules

- **Never leak vault contents.** Read `vault_get` / `vault_totp` and use the values inline. Don't echo passwords, API keys, recovery codes, or live TOTP codes back to the user unless they explicitly ask.
- **Match scopes to intent.** If a tool isn't available, the API key isn't scoped for it. Tell the user which scope they need (`mail:send`, `vault:write`, `payments:accept`, etc.) and link them to console.loomal.ai to add it. Don't try alternate tools to work around missing scope.
- **Reply, don't send, in existing threads.** `mail_reply` preserves `In-Reply-To` headers; `mail_send` starts a new thread.
- **One identity per task.** If a user has multiple Loomal identities (e.g., `loomal-sales`, `loomal-support`), each is a separate MCP server registration — confirm with `identity_whoami` which one you're operating as before sending mail or storing credentials.
- **Payments are real money.** `payments_redeem` settles USDC on Base mainnet. Confirm amount + resource before calling.

## Troubleshooting

- **Tool not found** → key is missing the scope. Check `identity_whoami.scopes`.
- **401 / invalid key** → key may be revoked. Rotate at console.loomal.ai.
- **Email not arriving** → SES inbound takes a few seconds; retry `mail_list_messages` after 5s. Check spam label.
- **TOTP code rejected** → server clock skew. Loomal uses RFC 6238 (30-second window); if the user's clock is off, codes won't match.

## Links

- Console: <https://console.loomal.ai>
- Docs: <https://docs.loomal.ai>
- API reference: <https://docs.loomal.ai/api-reference>
- MCP package: <https://www.npmjs.com/package/@loomal/mcp>
