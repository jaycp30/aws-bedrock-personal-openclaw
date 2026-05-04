# OpenAI / ChatGPT Plus Setup for OpenClaw

This guide covers how to connect OpenAI to your OpenClaw instance. There are two ways to do it depending on what you have.

**Option A - ChatGPT Plus subscription (OAuth, no API billing)**
**Option B - OpenAI API key (pay-per-token)**

---

## Background

In May 2026, OpenAI opened ChatGPT subscription access to OpenClaw. Sam Altman announced it on X:
> "you can sign in to openclaw with your chatgpt account now and use your subscription there! happy lobstering."

This means if you already pay $20/month for ChatGPT Plus, you can connect it to OpenClaw via OAuth and use GPT-5.5 at no additional API cost. No separate API key or billing required.

Reference: [OpenAI opens ChatGPT subscriptions to OpenClaw's 3.2M users — The Next Web](https://thenextweb.com/news/openai-openclaw-chatgpt-subscription-agent)

---

## Prerequisites

- OpenClaw 2026.3.13 or later installed and running
- **For Option A:** An active ChatGPT Plus, Pro, or Team subscription
- **For Option B:** An OpenAI API key from [platform.openai.com/api-keys](https://platform.openai.com/api-keys)

> **Important:** If you are on version 2026.3.24, update before proceeding. That version has a known bug where OAuth login succeeds but all model calls fail with `HTTP 401: Missing scopes: model.request`. Update first:
> ```bash
> sudo npm install -g openclaw@latest
> openclaw --version
> ```

---

## Option A: ChatGPT Plus Subscription (OAuth)

This is the recommended route if you have a ChatGPT Plus subscription. No API billing, no separate key.

### Step 1: Run the OAuth login

```bash
openclaw models auth login --provider openai-codex
```

OpenClaw will detect you are on a remote/VPS environment and show you a URL instead of opening a browser automatically.

### Step 2: Authorize in your local browser

Copy the `https://auth.openai.com/oauth/authorize?...` URL from the terminal and open it in your browser on your local machine. Sign in with the OpenAI account that has your ChatGPT Plus subscription and click **Continue** to authorize.

### Step 3: Paste the redirect URL back

After authorizing, your browser will redirect to `http://localhost:1455/auth/callback?code=...` which will show an error or blank page. That is expected. Copy the full URL from your browser's address bar (the entire thing including the `?code=...` part) and paste it back into the terminal prompt.

OpenClaw will exchange it for an access token and write the auth profile to `~/.openclaw/openclaw.json`.

### Step 4: Set the model

```bash
openclaw config set agents.defaults.model.primary openai-codex/gpt-5.5
```

### Step 5: Restart the gateway

```bash
systemctl --user restart openclaw-gateway
systemctl --user status openclaw-gateway
```

### Step 6: Verify

```bash
openclaw models list
```

You should see `openai-codex/gpt-5.5` as your active primary model.

---

## Option B: OpenAI API Key (Pay-per-token)

Use this if you do not have a ChatGPT Plus subscription, or if you need guaranteed uptime without weekly quota limits.

> **Note:** Your ChatGPT Plus subscription and your OpenAI API account are completely separate billing systems. API usage is billed on top of your Plus subscription — it does not draw from your Plus plan. Make sure you have credits loaded at [platform.openai.com](https://platform.openai.com).

### Step 1: Get your API key

Go to [platform.openai.com/api-keys](https://platform.openai.com/api-keys) and create a new secret key. Copy it immediately — you cannot view it again after closing the page.

### Step 2: Add the key to OpenClaw config

```bash
openclaw config set env.OPENAI_API_KEY sk-your-key-here
```

Or edit the config file directly:

```bash
nano ~/.openclaw/openclaw.json
```

Add the following under the top level:

```json
{
  "env": {
    "OPENAI_API_KEY": "sk-your-key-here"
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "openai/gpt-4o"
      }
    }
  },
  "gateway": {
    "port": 18789,
    "mode": "local"
  }
}
```

Use `openai/gpt-4o` for best capability or `openai/gpt-4o-mini` for lower cost.

### Step 3: Restart the gateway

```bash
systemctl --user restart openclaw-gateway
systemctl --user status openclaw-gateway
```

### Step 4: Verify

```bash
openclaw models list
```

---

## Troubleshooting

**OAuth login succeeds but model calls return 401 / Missing scopes: model.request**
You are on version 2026.3.24 which has this bug. Update OpenClaw and redo the login flow.
```bash
sudo npm install -g openclaw@latest
openclaw models auth login --provider openai-codex
```

**`--device-code` flag returns "unknown option"**
That flag was introduced in a version later than 2026.3.24. Update OpenClaw and the flag will be available, or just use the paste-redirect-URL flow without the flag as described in Option A above.

**ChatGPT Plus weekly quota hit**
ChatGPT Plus includes a weekly usage cap for Codex access. If you hit it mid-week, add a fallback model:
```bash
openclaw models fallbacks add openrouter/google/gemini-3-flash-preview
```

**Line plugin missing after update**
If you see `plugins.entries.line: plugin not found` after updating, reinstall it:
```bash
openclaw plugins install line
openclaw doctor --fix
systemctl --user restart openclaw-gateway
```

**Re-authenticate after token expiry**
OAuth tokens refresh automatically during active use. If a session expires, just re-run:
```bash
openclaw models auth login --provider openai-codex
```

---

## Cost Comparison

| Route | Cost | Quota |
|---|---|---|
| ChatGPT Plus OAuth | $20/month (existing subscription) | Weekly limit on Plus |
| ChatGPT Pro OAuth | $200/month (existing subscription) | Higher / unlimited |
| OpenAI API key | Pay-per-token, no flat fee | No weekly limit |

For personal use through Line, Telegram, or WhatsApp at moderate volume, the Plus OAuth route is the most cost-effective option.
