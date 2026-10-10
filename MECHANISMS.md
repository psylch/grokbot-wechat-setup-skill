# How the tested setup works (do not improvise)

Everything here is already implemented in `bridge/` and the routine prompts in `routines/`. A new bot should install these as-is rather than writing its own versions.

**Two automations, one program.** The forwarder (`bridge/forward.py`, kept alive by `run_forward.sh`) is a long-running program, not an automation. On top of it there are exactly two routines: the inbound Webhook routine (`routines/inbound-webhook.md`) and a 15-minute scheduled watchdog (`routines/watchdog.md`, runs `bridge/watchdog.sh`). The watchdog does not forward anything; it only checks the forwarder and restarts it.

**Batching.** Messages the user sends in a row are buffered in `pending.json` and posted as one batch after a 3-second pause (at most 60 s after the first). One batch means one agent run.

**Interruptible replies.** Each batch carries `trigger_ms`. The agent always replies with `send.py --reply-to-ms <trigger_ms>`. If the user wrote something newer meanwhile, `send.py` exits 4 and sends nothing; the newer run answers everything.

**Conversation history.** Every inbound batch and every successful outbound text (replies and app-side pushes) is appended to `history.log`. Each webhook POST includes its last 40 lines, so each fresh run can see what was just said on both sides.

**Context notes.** The forwarder adds `context_note` when the batch is a reply to the keep-alive nudge, or a bare "1" after 12+ hours of silence, so the agent just says "续上了" instead of asking what "1" means.

**Shared in-progress file (`active.md`).** History only has the words, not the task behind them. So whenever the main bot pushes a task-related WeChat message from the app side, it also adds or updates a line in `$BRIDGE/active.md` (time, item, link, owner, status, what it's waiting on). The inbound routine reads this file first and must not re-ask "who / which site" when it is clear, and writes back any new work it takes on. Save a standing memory on the main bot to keep this up. Template in `templates/active.md`.

**Push window and outbox.** WeChat only lets the bot push within about 24 h of the user's last message, and a refusal still returns HTTP 200 (check `ret`/`errcode`). Refused sends go to `outbox.json` (exit 3) and are flushed right after the user next writes. A single keep-alive nudge goes out 18–23 h after the last message, never 00:00–07:59 Beijing time; if that whole slot is quiet hours it is sent earlier, just before quiet hours.

**App-side pushes.** Other routines (daily report, reminders) push with `send.py --app "…"`, which prefixes `[grok bot] ` so the user can tell them apart from replies.

**Secrets.** `credentials.json` (iLink token) and the webhook key never go into chat or git. Collect the webhook key through Grok Bot's secure secret input. The webhook URL/key panels open only in the desktop app.
