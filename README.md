# SmsManager MCP Server

A [Model Context Protocol](https://modelcontextprotocol.io) server that lets an AI assistant
(Claude, Cursor, …) work with a [SmsManager](https://smsmanager.com) account on your behalf — send
messages, check delivery, manage API keys, top up credit, order senders, and check account status.

- **Endpoint:** `https://app-api.smsmanager.com/functions/v1/mcp`
- **Transport:** Streamable HTTP
- **Auth:** OAuth 2.1, per user — **no API keys to configure**
- **Interactive UI:** [MCP Apps](#interactive-ui-mcp-apps) — compose & send SMS, delivery status,
  order confirmation and billing forms right in the chat (Claude, ChatGPT, VS Code, …)

Every request acts as one signed-in SmsManager user, scoped to **one workspace** the user picks
when they connect.

Looking for the knowledge layer instead? The
[smsmanager-skills](https://github.com/smsmngr/smsmanager-skills) catalog teaches AI coding agents
the SmsManager APIs; this MCP server is the action layer that operates a real account.

---

## Connecting

### Claude (desktop / web — custom connector)

Add a custom connector with the URL above. Claude discovers the OAuth settings automatically;
accept the defaults:

- **Authentication:** *Sign in now* — the server requires a login.
- **OAuth client:** *Register automatically* (Dynamic Client Registration).

### Claude Code (CLI)

```bash
claude mcp add --transport http smsmanager https://app-api.smsmanager.com/functions/v1/mcp
```

Then run `/mcp` and choose to authenticate.

### Cursor

Install the [SmsManager plugin](https://github.com/smsmngr/smsmanager-cursor-plugin) (registers
the server automatically), click the button below, or add the server manually to `mcp.json`:

[![Add to Cursor](https://img.shields.io/badge/Add_to-Cursor-a81943)](cursor://anysphere.cursor-deeplink/mcp/install?name=smsmanager&config=eyJ1cmwiOiJodHRwczovL2FwcC1hcGkuc21zbWFuYWdlci5jb20vZnVuY3Rpb25zL3YxL21jcCJ9)

```json
{
  "mcpServers": {
    "smsmanager": {
      "url": "https://app-api.smsmanager.com/functions/v1/mcp"
    }
  }
}
```

### ChatGPT / Codex

Enable **Developer mode** in ChatGPT settings, then add a plugin with the server URL above —
sign in and pick a workspace, then invoke it by typing `@`. The
[SmsManager ChatGPT plugin](https://github.com/smsmngr/smsmanager-chatgpt-plugin) adds workflow
skills on top of the server.

### Other MCP clients

Any client that supports **Streamable HTTP** with **OAuth 2.1** (Dynamic Client Registration)
can connect with just the endpoint URL.

### What happens when you connect

1. The client opens a browser to sign in to SmsManager.
2. A **consent screen** shows the app requesting access and asks you to **pick a workspace**. The
   connection is bound to that workspace.
3. After you approve, the client is connected and the tools become available.

**Switching workspace:** disconnect / revoke the app in your MCP client and reconnect, then pick a
different workspace on the consent screen.

---

## How access works

- The token identifies **you** (a SmsManager user); the workspace you chose at consent time is
  remembered and every tool call operates on that workspace.
- The server only sees data you're already allowed to see — workspace access is re-checked on
  every request, so losing membership immediately cuts off access.
- Sending messages and ordering services **spend the workspace's credit** and use the workspace's
  API key server-side. The key itself is never exposed to the AI.

---

## Tools

Legend: 🔎 read-only · ✏️ writes/changes state · 💸 spends credit · 🖼️ opens an interactive view in
clients with [MCP Apps](#interactive-ui-mcp-apps) support

### Identity & account

#### `whoami` 🔎
Who you are and which workspace this connection uses. Includes a short account block (activated?,
review status, phone verified?, currency). No input.

#### `get_account_status` 🔎
Detailed activation state and the concrete next step to get activated: `review_status`
(`not_submitted` / `pending` / `approved` / `denied`), phone verification, and the current usage
description / sample message on file. No input.

> **Activation is finished in the browser.** Phone verification needs a code widget that can't be
> automated, so this tool reports status and requirements — it can't submit the verification
> itself. The assistant can help you *draft* the usage description (min 11 chars) and a sample
> message (min 6 chars), then you complete it at `https://app.smsmanager.com/app/account-verify`.

### Messaging

#### `compose_message` 🔎🖼️
Prepares an SMS draft for you to review — **sends nothing**. All inputs are optional prefills:
`to`, `text`, `sender`, `schedule_at`. Returns the draft plus the workspace's senders, credit and
prices.

In clients with MCP Apps support this opens the **compose form**: recipients (validated as you
type), text with a live character / SMS-part counter, sender picker, *Now / Later* with a
date-time picker, a cost estimate against your credit, and a *Transactional* switch. You press
**Send** (or **Schedule**) yourself — the assistant is told it was sent and won't send it again.
Clients without the UI get the draft as text; the assistant confirms it with you and calls
`send_message`.

#### `send_message` ✏️💸🖼️
Send a real SMS to up to 10 recipients — now or later.
- `to` — array of phone numbers, E.164 **without** the leading `+` (e.g. `420777123456`), 1–10 entries
- `text` — message body, 1–1000 chars
- `sender` *(optional)* — a registered sender ID; omit to use the workspace default
- `schedule_at` *(optional)* — send later: ISO 8601 with a timezone offset, e.g.
  `2026-10-01T09:00:00+02:00` (1 minute to 365 days ahead). Credit is charged at send time.
- `tag` *(optional)* — comma-separated tags to group messages; `transactional` and `priority` are
  special. Untagged messages are `promotional` and checked against the opt-out list.

Returns a `request_id`, `scheduled_at` and `recipients` — each number with its `status`
(`accepted` / `rejected`) and `message_id`. A `rejected` recipient was **not** sent (bad number or
no credit).

#### `get_message_status` 🔎🖼️
Delivery status of one message.
- `message_id` — from a send result

`delivery_code` `201` = delivered, `202` = seen. An unknown id returns `message: null` (not an
error).

#### `list_messages` 🔎🖼️
Sent messages for one day (paginated).
- `date` — `YYYY-MM-DD`
- `tag`, `phone_number` *(optional)* — filters
- `cursor` *(optional)* — pass back the returned `next_cursor` for the next page

#### `list_inbox` 🔎
Received (inbound) messages for one day.
- `date` — `YYYY-MM-DD`

### API keys

#### `list_apikeys` 🔎
Lists the workspace's child API keys ("apps") with their notes, scopes and status.

#### `create_child_apikey` ✏️
Creates a new child API key for an integration and returns it.
- `note` — label for the key (the app/integration name)
- `scope` *(optional)* — `api` (send/read only, default) or `full` (all permissions)
- `share_senders` *(optional)* — whether the key may use the workspace's senders (default `true`)
- `currency` *(optional)* — `CZK` or `EUR`; defaults to the workspace currency

> The returned key is a live credential — use it for the integration; don't reuse the workspace's
> own key.

### Credit & billing

#### `get_credit` 🔎
Current credit balance and account currency. No input.

#### `get_billing_details` 🔎🖼️
Invoicing details on file (name, address, company id, VAT). Empty fields mean billing isn't set up
yet. **Billing must be filled before credit can be topped up.** With MCP Apps this is an editable
form that saves directly and can show the bank-transfer top-up details.

#### `set_billing_details` ✏️
Fills or updates invoicing details. Pass only the fields you want to change — existing values are
preserved (read-merge-write).
- `name`, `email`, `street`, `city`, `zip`
- `country` — ISO 3166-1 alpha-2 (e.g. `CZ`)
- `company_id` — registration number (IČO)
- `company_vat` — VAT number (DIČ)

#### `get_topup_details` 🔎
Bank-transfer payment instructions for adding credit — bank account number, IBAN, variable symbol
(VS). You make the transfer yourself; credit is added once the payment is processed. (Card top-up
is done in the dashboard.) No input.

### Services (senders, numbers)

#### `list_services` 🔎
Catalog of orderable paid services (alphanumeric sender IDs, dedicated virtual numbers, SIM
hosting, …) with setup and monthly fees per currency. Use it to find a `service_id`. No input.

#### `order_service` ✏️💸🖼️
Order a paid service. **Spends credit.**
- `service_id` — from `list_services`, e.g. `sender_sms_420_alnum`
- `period` *(optional)* — `once` / `month` / `quarter` / `year` (the service minimum may override)
- `subject` *(optional)* — for "defined" services, the sender text or number; omit for pool-assigned ones
- `payload` *(optional)* — service-specific fields, e.g. `{ "country": "CZ", "proofName": "…" }`
- `confirm` *(optional, default `false`)* — **`false` = dry-run quote (no charge); `true` = place the real order**

Always call with `confirm: false` first to get the exact `total_fee`, confirm it, then call again
with `confirm: true`. With MCP Apps the quote is a card with a **Confirm and pay** button that
places exactly the quoted order.

#### `cancel_service` ✏️
Flags a paid service to cancel at the end of the paid period (no refund; stays active until
`active_until`). Idempotent.
- `service_id` — e.g. `sender_sms_420_alnum`
- `subject` — the exact subject the service was ordered with

---

## Common workflows

**Send and confirm delivery**
`compose_message` (you fill in and send the form) — or `send_message` directly → note the
`message_id` → `get_message_status` (or `list_messages` for a day's overview).

**Schedule a message**
`compose_message` → pick *Later* in the form — or `send_message` with `schedule_at`.

**Top up credit (bank transfer)**
`get_billing_details` → if empty, `set_billing_details` → `get_topup_details` → make the transfer
with the given VS.

**Order an alphanumeric sender**
`list_services` (find the `service_id`) → `order_service` with `confirm: false` (review the fee) →
`order_service` with `confirm: true`.

**Get the account activated**
`get_account_status` → the assistant helps draft the usage description + sample message → finish
phone verification and submit at `https://app.smsmanager.com/app/account-verify`.

**Connect a new integration**
`create_child_apikey` → use the returned key in the integration.

---

## Interactive UI (MCP Apps)

The server implements the [MCP Apps](https://github.com/modelcontextprotocol/ext-apps) extension
(`io.modelcontextprotocol/ui`). Tools marked 🖼️ link to the `ui://smsmanager/app.html` resource,
which supporting clients render in a sandboxed frame inside the conversation:

| Tool | View |
|------|------|
| `compose_message` | Compose & send form (recipients, text, sender, schedule, cost) |
| `send_message`, `get_message_status`, `list_messages` | Delivery status with *Refresh* |
| `order_service` (quote) | Order confirmation card |
| `get_billing_details` | Billing form + top-up details |

- **Supported clients:** Claude (web, desktop, mobile), ChatGPT, VS Code, Goose and other MCP Apps
  hosts. Clients without it (e.g. Claude Code, Cursor) get the same tools with text results.
- **Security:** the view makes no network requests of its own. It only receives tool results and
  acts through tool calls the client forwards to this server, so your API key and login token
  never reach it. Sending, ordering and saving always need an explicit click.

---

## Protocol compatibility

- **MCP spec:** 2025-11-25 and earlier (TypeScript SDK 1.30), stateless Streamable HTTP with JSON
  responses, OAuth 2.1 resource server with protected-resource metadata and Dynamic Client
  Registration. Tools return `structuredContent` alongside text, and the send/compose tools
  declare an `outputSchema`.
- **"MCP 2.0":** there is no spec with that name — spec revisions are dated. The newest,
  **2026-07-28** (stateless handshake, multi round-trip requests), will be adopted together with
  the v2 TypeScript SDK once major clients support it; the server is already stateless and
  session-free, which is what that revision requires.
- **Extensions:** MCP Apps (`io.modelcontextprotocol/ui`, stable spec 2026-01-26).

---

## Limitations

- One workspace per connection; switching means reconnecting.
- Up to 10 recipients per `send_message` call (and per compose form).
- Account activation (phone verification, review submission) is completed in the browser — the
  tools read status and help prepare, but can't submit it.
- `get_topup_details` returns bank-transfer instructions only; card top-up is in the dashboard.

---

## Related

- [smsmanager-skills](https://github.com/smsmngr/smsmanager-skills) — agent skills for the
  SmsManager APIs (Claude Code plugin, `npx skills add`).
- [smsmanager-cursor-plugin](https://github.com/smsmngr/smsmanager-cursor-plugin) — Cursor plugin
  bundling the skills and this MCP server.
- [smsmanager-chatgpt-plugin](https://github.com/smsmngr/smsmanager-chatgpt-plugin) — ChatGPT/Codex
  plugin bundling workflow skills and this MCP server.
- Developer docs: https://smsmanager.com/docs (Czech: https://smsmanager.cz/docs)

## License

[Apache-2.0](LICENSE)
