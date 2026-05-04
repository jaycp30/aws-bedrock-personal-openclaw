# Using Your Own Claude Account with OpenClaw

How to configure OpenClaw to use your personal Anthropic credentials instead of AWS Bedrock. Useful when Bedrock quotas are pending approval or you prefer to bill through Anthropic directly.

---

## Know the Difference First

There are two separate paths here and they look similar but they are not the same thing.

| | claude.ai subscription (setup-token) | Anthropic API key |
|---|---|---|
| Where to get it | claude.ai (your existing account) | console.anthropic.com (separate account) |
| Billing | Draws from "extra usage" credits, not your plan | Pay per token |
| Rate limits | Subject to extra usage credit balance | Based on API tier |
| Best for | Testing, short-term workaround | Always-on persistent setup |

For a server running 24/7 and handling LINE webhooks, the **Anthropic API key** is the better choice. The subscription approach works but your bot and your personal Claude usage compete for the same extra usage credit pool.

---

## Option 1: claude.ai Subscription (setup-token method)

This connects OpenClaw to your existing claude.ai account using a temporary setup token.

**Important caveat from April 4, 2026 onward:** Anthropic now routes third-party app usage to a separate "extra usage" bucket, not your plan limits. If that bucket is empty, you will hit this error:

```
LLM request rejected: Third-party apps now draw from your extra usage,
not your plan limits. Add more at claude.ai/settings/usage and keep going.
```

To fix it: go to [claude.ai/settings/usage](https://claude.ai/settings/usage) and add extra usage credits. Your Pro plan subscription credits do not cover this.

### Setup Procedure

**Step 1:** SSM into the EC2 instance and stop the gateway before configuring:

```bash
systemctl --user stop openclaw-gateway.service
openclaw configure --section model
```

**Step 2:** Walk through the configure wizard prompts:
- Where will the Gateway run? → **Local (this machine)**
- Model/auth provider? → **Anthropic**
- Anthropic auth method? → **Anthropic token (paste setup-token)**

**Step 3:** On your local machine in a separate terminal, generate the setup token:

```bash
claude setup-token
```

Copy the token it outputs. It starts with `sk-ant-oat01-...`.

**Step 4:** Paste the token into the configure wizard prompt. You can give it a name (e.g. your email) or leave it blank.

**Step 5:** After the wizard completes, restart the gateway:

```bash
systemctl --user restart openclaw-gateway.service
sleep 5
ss -tlnp | grep 18789
```

**Step 6:** The configure wizard may have regenerated the dashboard access token. Grab it before reconnecting:

```bash
grep '"token"' ~/.openclaw/openclaw.json
```

**Step 7:** Restart port forwarding on your local machine and open the dashboard with the token from Step 6.

---

## Option 2: Anthropic API Key (pay-per-token)

This uses a proper API key from Anthropic's developer platform - separate from your claude.ai account.

### Get Your API Key

1. Go to [console.anthropic.com](https://console.anthropic.com) (sign up if you haven't - it's separate from claude.ai)
2. Add a credit card under **Billing**
3. Generate an API key under **API Keys**
4. Copy the key - it starts with `sk-ant-api03-...`

To reach **Tier 2** rate limits, you need to have purchased a cumulative total of $40 in API credits. After that, your account upgrades automatically and monthly limits increase significantly.

### Setup Procedure

**Step 1:** SSM into the EC2 instance and stop the gateway:

```bash
systemctl --user stop openclaw-gateway.service
openclaw configure --section model
```

**Step 2:** Walk through the configure wizard:
- Where will the Gateway run? → **Local (this machine)**
- Model/auth provider? → **Anthropic**
- Anthropic auth method? → **API key**

**Step 3:** Paste your API key when prompted.

**Step 4:** Restart and verify:

```bash
systemctl --user restart openclaw-gateway.service
sleep 5
ss -tlnp | grep 18789
```

**Step 5:** Grab the (possibly regenerated) dashboard token and reconnect:

```bash
grep '"token"' ~/.openclaw/openclaw.json
```

---

## Switching Models After Setup

Once either option is configured, you can change the Claude model without re-running the full wizard. Edit the config directly:

```bash
sed -i 's|"primary":.*|"primary": "anthropic/claude-haiku-4-5-20251001-v1:0"|' ~/.openclaw/openclaw.json
```

Other valid model strings:
- `anthropic/claude-sonnet-4-6`
- `anthropic/claude-sonnet-4-5-20250929-v1:0`
- `anthropic/claude-haiku-4-5-20251001-v1:0`

Wait about 30 seconds after editing, then refresh the dashboard. The change takes effect via hot-reload without a manual restart.

---

## Known Issues

**Dashboard disconnects after running the configure wizard**

This is expected. The configure wizard rewrites `openclaw.json`, which triggers OpenClaw's hot-reload mechanism (SIGUSR1). The gateway restarts, which drops the port forwarding tunnel and your dashboard connection.

Fix: after the wizard finishes, restart the gateway and reconnect the tunnel. Always grab a fresh token from `openclaw.json` after a configure run since it may have changed.

**"Unit openclaw-gateway.service could not be found"**

You ran the systemctl command without the `--user` flag. OpenClaw runs as a systemd user service under the `ubuntu` account, not a system-level service. Always include `--user`:

```bash
systemctl --user restart openclaw-gateway.service
systemctl --user status openclaw-gateway.service
journalctl --user -u openclaw-gateway.service -n 30 --no-pager
```

---

## When to Switch Back to Bedrock

If your AWS Bedrock quotas come through or get approved, switch back. Your v1.3 CloudFormation stack is already fully wired for Bedrock - the IAM user, SSM credentials, and `auth-profiles.json` are all in place. You just need to update the model in the config:

```bash
sed -i 's|"primary":.*|"primary": "amazon-bedrock/global.anthropic.claude-sonnet-4-6"|' ~/.openclaw/openclaw.json
```

Bedrock is the cleaner long-term path for a persistent server deployment. There is no subscription tied to a personal account, no extra usage credits to monitor, and billing stays within AWS.
