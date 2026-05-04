## ⚠️ Token Usage Warning

OpenClaw is an AI agent, not a chat interface. This distinction matters for cost and quota.

When you send a message directly on ChatGPT or Claude.ai, it is one request. When you send the same message through OpenClaw, it can trigger 5-10 API calls behind the scenes — tool lookups, context loads, reasoning steps, and revisions — and every single one of those calls re-sends your entire conversation history. The longer a session runs, the heavier each new message becomes.

**If you are on a flat subscription like ChatGPT Plus OAuth, this eats your weekly quota faster than direct chat would. If you are on a pay-per-token API key, this directly increases your bill.**

---

### Why Token Bloat Happens

**Context accumulation** is the biggest driver. OpenClaw stores session history in `.jsonl` files under `~/.openclaw/agents/main/sessions/`. Every message you send in an active session appends to that file, and every subsequent message re-sends the whole thing as context. A session with 35 messages can grow to 2-3MB of context being sent on every single call.

**Tool outputs** make it worse. When OpenClaw calls a tool — web search, file read, memory lookup — the output gets stored in the session and carried forward. A single config schema fetch can inject hundreds of kilobytes of JSON into your session permanently.

**Reasoning mode** multiplies everything. If thinking or reasoning mode is enabled, the model generates internal reasoning chains before answering, which can increase token usage 10-50x per request.

---

### How to Minimize Token Consumption

**Start a new session regularly**

This is the single most effective habit. Instead of letting one conversation run for days, start fresh often. Send this in chat:

```
/new
```

This clears accumulated context and resets the session. Do this at the start of each new topic or task.

**Compact long sessions before they grow too large**

If a session has been running a while, compact it before continuing:

```
/compact
```

OpenClaw summarizes the conversation history into a shorter form and replaces the full transcript with the summary, reducing what gets re-sent on future calls.

**Check your session sizes periodically**

SSH into your instance and check if any sessions have grown unusually large:

```bash
du -h ~/.openclaw/agents/main/sessions/*.jsonl | sort -h
```

Anything over 500KB is worth investigating. You can delete stale sessions manually — just make sure the gateway is not actively using them.

**Limit the context window in config**

Add a `contextTokens` cap to your `~/.openclaw/openclaw.json` to prevent any single session from using more than a set number of tokens:

```json
{
  "agents": {
    "defaults": {
      "contextTokens": 50000
    }
  }
}
```

Adjust the number based on your needs. 50,000 is a reasonable ceiling for general chat use.

**Disable reasoning/thinking mode if you do not need it**

Reasoning mode is powerful but expensive. Unless you are doing complex multi-step tasks, keep it off:

```json
{
  "agents": {
    "defaults": {
      "models": {
        "openai-codex/gpt-5.5": {
          "params": {
            "thinking": {
              "type": "disabled"
            }
          }
        }
      }
    }
  }
}
```

**Use a lighter model for simple tasks**

Not every message needs the most capable model. For routine questions and simple chat, switch to a smaller model mid-session:

```
/model openai/gpt-4o-mini
```

Switch back to your primary when you need it.

**Set up a fallback model**

If you hit your weekly quota, OpenClaw will go silent unless you have a fallback configured. Add one so the bot keeps responding automatically:

```bash
openclaw models fallbacks add openrouter/google/gemini-3-flash-preview
```

---

### Quick Reference

| Action | Command |
|---|---|
| Start a new session | `/new` |
| Compact current session | `/compact` |
| Check current model | `/model status` |
| Switch to lighter model | `/model openai/gpt-4o-mini` |
| Check session file sizes | `du -h ~/.openclaw/agents/main/sessions/*.jsonl \| sort -h` |
| Restart gateway | `systemctl --user restart openclaw-gateway` |

---

> **Bottom line:** For casual personal use through Line or Telegram, OpenClaw's token overhead is manageable. Where people get into trouble is letting sessions run for days without resetting, or running automated agents doing complex multi-step tasks continuously. The `/new` and `/compact` commands are your best tools for keeping usage under control.
