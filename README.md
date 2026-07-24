# Telegram Public Channel Monitor — n8n companion workflow

Optional **n8n** companion workflow for the Apify Actor `bgy_1203/telegram-public-channel-monitor`.

The Actor is **stateless** and fetches **one public Telegram channel per run**. This workflow wraps
it in n8n to add — as workflow features, not Actor features:

- **Multi-channel scheduling** — polls a list of channels on an interval (default every 15 minutes).
- **Persistent seen-post state** — remembers seen message-ID ranges per channel across runs.
- **Keyword filtering** — `any` / `all` / `off`, case-sensitive or not.
- **Duplicate prevention** — already-seen posts are not alerted again across runs.
- **Telegram alerts** — new matching posts sent to your Telegram chat via your own bot.

## Quickstart
1. Import the workflow JSON into n8n.
2. Attach your **Apify** (HTTP Bearer) and **Telegram Bot** credentials.
3. Edit the `CONFIG` block (channels, keywords, `alertChatId`) and **activate**.

- [Setup guide](./SETUP-GUIDE.md)
- [Workflow JSON](./workflow/telegram-public-channel-monitor.workflow.json)

## Boundaries
Public `t.me/s/` channels only. One channel per Actor run. No Telegram login or session, no private
channels, no member/contact scraping, no access-control bypass. All monitoring state, filtering, and
alerting live in this workflow — the Actor itself stays stateless.
