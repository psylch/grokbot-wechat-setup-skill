# grokbot-wechat-setup-skill

A Grok Bot skill that walks a new user through connecting WeChat to their Grok Bot: scan-to-bind with the QR sent in chat, inbound webhook setup (desktop only), forwarder, watchdog, and a plain-language wrap-up.

Skill file: `wechat-grok-bot-config/SKILL.md`

## Background a new bot needs

- **What it is.** WeChat's official ClawBot feature lets a WeChat account talk to a bot over Tencent's iLink bot API. A small forwarder on the bot's computer long-polls iLink, batches the user's messages and POSTs them to a Grok Bot webhook routine. The bot replies with `send.py`.
- **Bridge code and CLI (required).** [psylch/grokbot-wechat](https://github.com/psylch/grokbot-wechat): `bind.py` (scan-to-bind, writes `credentials.json`), `forward.py` + `run_forward.sh` (forwarder), `send.py` (reply / push CLI, exit 3 = queued, exit 4 = superseded), `watchdog.sh`, and routine templates in `routines/` (`inbound-webhook.md`, `watchdog.md`). Its README covers the architecture, payload format, 24h push window and known pitfalls.
- **iLink protocol write-up.** [WeChat ClawBot iLink protocol (chihao.li)](https://www.chihao.li/blog/en/wechat-clawbot-ilink-protocol)
- **Tencent's reference client.** [@tencent-weixin/openclaw-weixin on npm](https://www.npmjs.com/package/@tencent-weixin/openclaw-weixin)

## Requirements

- A WeChat account with ClawBot available (rolling out gradually; at release roughly iOS WeChat 8.0.70+). Don't ask the user about this up front; only explain it if scanning fails.
- Grok Bot desktop app for copying the webhook URL and key (the mobile app can't open those panels yet).
- Python 3.9+ on the machine running the forwarder (Grok Bot's own computer by default).

## Install the skill

Copy `wechat-grok-bot-config/` into your Grok Bot skills, or ask your bot to save it as a skill from this repo.
