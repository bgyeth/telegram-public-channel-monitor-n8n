# Setup guide — Telegram Public Channel Monitor (n8n workflow)

A ready-made **n8n** workflow that turns the **Telegram Public Channel Monitor** Apify Actor
(`bgy_1203/telegram-public-channel-monitor`) into a continuous, multi-channel monitor with
new-post detection, keyword filtering, and Telegram alerts. The Actor stays **stateless** and
reads **one public channel per run**; this workflow adds the scheduling, state, filtering, and
alerting on top.

## What the workflow does (these are n8n-workflow features, not Actor features)
- **Multi-channel monitoring** — loops your channel list, calling the Actor once per channel.
- **Persistent new-post detection** — remembers seen message-ID ranges per channel across runs
  (n8n workflow static data), so already-seen posts are not alerted again across runs.
- **Keyword filtering** — `any` / `all` / `off`, case-sensitive or not.
- **Cross-run duplicate prevention** — per-channel, independent of the Actor's within-run dedup.
- **Telegram alerts** — new matching posts sent to your Telegram chat via your own bot.
- **Scheduled monitoring** — default every 15 minutes.

## Prerequisites
- An **n8n** instance (self-hosted or n8n Cloud). Uses core nodes only: Schedule Trigger, Code,
  HTTP Request, IF, Telegram.
- An **Apify API token** (Apify Console → Settings → Integrations → API tokens).
- A **Telegram bot** (create via **@BotFather**) and the **chat ID** you want alerts sent to.

## 1) Import
Download `telegram-public-channel-monitor.workflow.json` and import it into n8n
(**Workflows → Import from File**, or paste the JSON).

## 2) Add credentials — never put tokens in the JSON
- **Run Actor Once Per Channel** (HTTP Request): attach a **Bearer Auth** (HTTP Bearer) credential
  whose token is your **Apify API token**.
- **Send Telegram Alert** (Telegram): attach a **Telegram Bot** credential with your bot token.

## 3) Configure
Edit **only** the `CONFIG` object inside the **Configuration & Channels** node:

| Field | What to set |
|---|---|
| `channels` | Public usernames, no `@`/URL/private, e.g. `['telegram','durov']` (validated `^[A-Za-z][A-Za-z0-9_]{4,31}$`) |
| `keywords` + `keywordMode` | `'any'`, `'all'`, or `'off'`; empty + `'off'` = alert on every new post |
| `caseSensitive` | `true` / `false` |
| `alertChatId` | Your Telegram chat ID (replace `REPLACE_WITH_TELEGRAM_CHAT_ID`) |
| `limit` | 1–500 recent messages per run (**default 5**) — drives cost; raise only if you see `WINDOW_SATURATED` |
| `firstRunMode` | `'baseline'` (recommended: first cycle sets baseline, no alerts) or `'alert'` |
| `actorMemoryMb` / `actorTimeoutSeconds` / `maxAlertTextChars` | Defaults are fine (128 MB / 120 s / 3000) |
| `actorBuild` | Pins the Actor build (default `'0.0.4'`; runtime identical to `0.0.5`, which only adds docs) |

## 4) Activate
**Save and activate** the workflow. Persistent state is saved **only for successful production
executions of an active workflow** — manual/editor test runs do **not** persist state.

## How the persistent state works (read this)
- The first successful non-empty cycle creates a **baseline** and sends **no alerts** (with
  `firstRunMode: 'baseline'`).
- State is tracked **per channel** as compressed message-ID ranges.
- Keep the schedule interval **longer than the worst-case run time**; executions are serialized
  (batch size 1).
- If a run returns exactly `limit` results, the status output flags `WINDOW_SATURATED` — raise
  `limit` or shorten the interval so you don't miss posts.
- Don't reset state casually — it can replay old posts.

## Cost (Apify pay-per-event) — driven by `limit`
You are billed **per message the Actor returns each run**, not per *new* message (n8n filters
duplicates **after** the Actor has already returned + billed them).

- Per run per channel: up to `limit` × **$0.001** (`apify-default-dataset-item`) + **$0.00005**
  (`apify-actor-start`).
- **The default `limit: 5` keeps this safe:** one channel every 15 min ≈
  `5 × $0.001 × 96 ≈ $0.48/day` + tiny start fees.
- **Raise `limit` only when you see `WINDOW_SATURATED`** (the poll window was too small and you may
  have missed posts between cycles). Raise it gradually — cost scales linearly.
- Caution: a high `limit` on a frequent schedule adds up fast — e.g. `limit: 100`, 3 channels,
  every 15 min ≈ **$28.8/day**.

## Boundaries (unchanged)
The Actor reads only public `t.me/s/` previews, **one channel per run**, with **no Telegram
login/session, no private channels, and no member/contact scraping**. All monitoring **state,
filtering, and alerting live in this n8n workflow**, not in the Actor.

## Troubleshooting
- No alerts on the first run → expected (baseline). New posts alert from the next cycle.
- Never any alerts → check the Telegram bot credential and `alertChatId` (the bot must be able to
  message that chat).
- Missing posts → `WINDOW_SATURATED`: increase `limit` or shorten the interval.
- State not persisting → the workflow must be **active** and run in **production** (not manual).

## Optional AI (not included)
The base workflow is deterministic substring matching — no AI. To add AI later, insert a
classifier on a **separate, explicitly enabled branch** after `Is New Matching Post?` so AI cost
or failures never block the base keyword path.
