# grokbot-wechat-setup-skill

A Grok Bot skill plus the exact, tested code and routine prompts for connecting WeChat (ClawBot / iLink) to a Grok Bot. A new bot should install what's here as-is, not write its own bridge.

## What's in here

- `wechat-grok-bot-config/SKILL.md`: the step-by-step user flow (scan-to-bind with the QR sent in chat, desktop-only webhook setup, self-test, forwarder, watchdog, plain-language wrap-up).
- `bridge/`: the bridge program and CLI, vendored from [psylch/grokbot-wechat](https://github.com/psylch/grokbot-wechat) at commit `9dfedc9`. `bind.py` binds, `forward.py` + `run_forward.sh` forward, `send.py` replies and pushes (exit 3 queued, exit 4 superseded), `watchdog.sh` checks and restarts. See `bridge/README.md` for architecture, payload format and pitfalls.
- `routines/`: the two routine prompts in use. `inbound-webhook.md` (Webhook trigger, includes the active.md context rule) and `watchdog.md` (every 15 minutes).
- `templates/active.md`: the shared in-progress file that keeps app-side pushes and WeChat-triggered runs on the same page.
- `MECHANISMS.md`: how batching, interruptible replies, history, context notes, active.md, the push window and outbox fit together.

## Background

- [WeChat ClawBot iLink protocol write-up](https://www.chihao.li/blog/en/wechat-clawbot-ilink-protocol)
- [Tencent's reference client @tencent-weixin/openclaw-weixin](https://www.npmjs.com/package/@tencent-weixin/openclaw-weixin)

## Requirements

- A WeChat account with ClawBot available (rolling out gradually). Don't ask the user up front; only explain if scanning fails.
- Grok Bot desktop app for copying the webhook URL and key (the mobile app can't open those panels yet).
- Python 3.9+ on the machine running the forwarder (the bot's own computer by default).
