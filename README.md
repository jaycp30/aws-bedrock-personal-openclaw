# OpenClaw on AWS with Amazon Bedrock

A self-hosted personal AI assistant running on AWS EC2, powered by Amazon Bedrock - no API keys, no local compute, no laptop dependency.

Based on the guide by [Jon Bonso / The Cloud Dojo](https://www.linkedin.com/pulse/stop-running-openclaw-your-laptop-how-i-built-secure-personal-bonso-thyyc/).

---

## What This Is

[OpenClaw](https://openclaw.ai) is an open-source AI agent framework. Instead of just chatting with an AI, it can take actions - respond to messages on WhatsApp/Telegram/Discord, run scheduled tasks, maintain memory across conversations, and use tools and skills.

The problem with running it locally: it needs to be on 24/7, has full access to your machine, and dies when your laptop sleeps or your Wi-Fi drops.

This deployment moves it to AWS, where it runs continuously on EC2 and calls Amazon Bedrock for its AI model instead of managing separate API keys per provider.

---

## Repository Structure

```
.
├── README.md
└── cloudformation/
    └── openclaw-bedrock.yaml
```

---

## Architecture

```
Your Phone (WhatsApp/Telegram)
        |
        v
  Messaging Platform
        |
        v
  OpenClaw Gateway (EC2 - Ubuntu 24.04, Graviton ARM64)
        |
        v
  Amazon Bedrock (Nova Lite / Nova Micro)
        |
        v
  AI Response
```

**Key components:**

- **EC2 Instance** - runs the OpenClaw gateway process (t4g.small, ARM64/Graviton)
- **Amazon Bedrock** - provides the AI model via unified API, no separate API keys needed
- **IAM Role** - EC2 authenticates to Bedrock via instance profile, no credentials stored anywhere
- **EBS Data Volume** - 30GB encrypted volume for OpenClaw config and data, survives instance stop/start
- **SSM Session Manager** - access to the instance and port forwarding, no SSH port open
- **VPC** - dedicated network with optional private endpoints so traffic never leaves AWS

---

## Why Bedrock Instead of Direct API Keys

| Feature | Standalone OpenClaw | OpenClaw + Bedrock |
|---|---|---|
| Setup | Manual CLI + Docker | 1-click CloudFormation |
| AI Model Access | Separate API keys per provider | Single unified API |
| Billing | Multiple bills (OpenAI, Anthropic, etc.) | Single AWS bill |
| Security | API keys in local files | IAM Roles, no keys stored |
| Networking | Public internet | Private VPC |
| Reliability | Depends on laptop uptime | 99.99% AWS uptime |

---

## Deployment

### Prerequisites

- AWS account with Bedrock model access enabled
- AWS CLI installed and configured locally
- SSM Session Manager plugin installed locally

### Step 1: Enable Bedrock Model Access

Before deploying, go to **AWS Console > Bedrock > Model access** and enable the models you want. The default is Amazon Nova Lite. This step is not in the original guide but the deployment will silently fail without it - the stack completes successfully but the AI never responds.

### Step 2: Deploy via CloudFormation

Use the template in [`cloudformation/openclaw-bedrock.yaml`](cloudformation/openclaw-bedrock.yaml) or from [aws-samples/sample-OpenClaw-on-AWS-with-Bedrock](https://github.com/aws-samples/sample-OpenClaw-on-AWS-with-Bedrock).

Key parameters to configure:

| Parameter | Recommended (Sandbox) | Notes |
|---|---|---|
| OpenClawModel | `global.amazon.nova-2-lite-v1:0` | Cheapest, good for everyday tasks |
| OpenClawVersion | `2026.3.24` | More stable, no extra approval needed |
| InstanceType | `t4g.small` | ARM64 Graviton, cheapest viable option |
| CreateVPCEndpoints | `false` | Saves ~$29/month for sandbox use |
| EnableDataProtection | `false` | Set to `true` for production |

Scroll to the bottom, check **"I acknowledge that AWS CloudFormation might create IAM resources"**, and click **Create stack**.

Wait ~8-10 minutes for `CREATE_COMPLETE`.

### Step 3: Fix the Bonjour Plugin Bug

**Do this before anything else.** This is a critical issue when running OpenClaw on AWS that the original guide does not mention.

The `bonjour` plugin handles local network discovery and requires multicast networking, which AWS VPC does not support. It crashes the gateway every ~45 seconds, putting it in a permanent crash loop. The service will appear to start successfully but port 18789 will never stay open long enough to accept connections.

Connect to the instance via SSM:

```bash
aws ssm start-session --target <instance-id> --region <region>
sudo su - ubuntu
```

Disable the bonjour plugin:

```bash
# Back up config first
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak

python3 -c "
import json
with open('/home/ubuntu/.openclaw/openclaw.json', 'r') as f:
    config = json.load(f)
if 'plugins' not in config:
    config['plugins'] = {}
if 'entries' not in config['plugins']:
    config['plugins']['entries'] = {}
config['plugins']['entries']['bonjour'] = {'enabled': False}
with open('/home/ubuntu/.openclaw/openclaw.json', 'w') as f:
    json.dump(config, f, indent=2)
print('Done')
"
```

Verify the change:

```bash
cat ~/.openclaw/openclaw.json | python3 -m json.tool | grep -A3 bonjour
```

Restart the gateway and confirm it stays up:

```bash
systemctl --user restart openclaw-gateway.service
sleep 15
ss -tlnp | grep 18789
```

You should see port 18789 in LISTEN state. If it is there after 15 seconds, the fix worked.

### Step 4: Access the Dashboard

The CloudFormation Outputs tab gives you everything you need under Steps 1-4.

**Port forwarding** - run on your local machine and keep the terminal open:

```bash
aws ssm start-session \
  --target <instance-id> \
  --region <region> \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["18789"],"localPortNumber":["18789"]}'
```

Then open the URL from the `Step3AccessURL` output in your browser. The token is already embedded in the URL.

**If you ever lose the token**, retrieve it from SSM Parameter Store:

```bash
aws ssm get-parameter \
  --name "/openclaw/<stack-name>/gateway-token" \
  --with-decryption \
  --region <region> \
  --query Parameter.Value \
  --output text
```

### Step 5: Connect a Messaging Channel

Once the dashboard is up, go to **Channels > Add Channel** and connect Telegram, WhatsApp, Discord, or Slack. The assistant will walk you through setup on first conversation.

---

## Estimated Cost

| Resource | Monthly Cost |
|---|---|
| EC2 t4g.small | ~$12-15 |
| EBS 30GB gp3 | ~$2.40 |
| VPC Endpoints (if enabled) | ~$29 |
| Bedrock | Pay per token used |
| **Total (no endpoints)** | **~$15-20/month** |

For sandbox use, keep `CreateVPCEndpoints=false`. For production, enable them so your traffic stays private within AWS.

**Tip:** Stop the EC2 instance when not in use. Your config persists on the separate EBS data volume. You only pay for EBS storage while stopped, not compute.

---

## Troubleshooting

### Gateway crash loop - bonjour plugin

**Symptom:** SSM port forwarding connects but immediately shows `Connection to destination port failed`. The gateway appears to start but dies after ~45 seconds.

**Diagnosis:** Check the journal logs:

```bash
journalctl --user -u openclaw-gateway.service --since "5 minutes ago" --no-pager
```

If you see this repeating:

```
[openclaw] Unhandled promise rejection: CIAO PROBING CANCELLED
openclaw-gateway.service: Main process exited, code=exited, status=1/FAILURE
```

That is the bonjour crash. Apply the fix in Step 3 above.

**Root cause:** Bonjour (mDNS) requires multicast networking for local device discovery. AWS VPC does not support multicast. The plugin gets stuck in a probing loop, gives up, and throws an unhandled rejection that kills the entire Node.js process. It restarts via systemd and crashes again on a ~45 second cycle indefinitely.

---

### AI not responding in the dashboard

**Symptom:** Messages send successfully in the UI but no response comes back.

**Diagnosis:** Test Bedrock connectivity directly from the EC2 instance:

```bash
MODEL_ID="global.amazon.nova-2-lite-v1:0"
aws bedrock-runtime invoke-model \
  --region <region> \
  --model-id "$MODEL_ID" \
  --body '{"messages":[{"role":"user","content":[{"text":"say hello"}]}],"inferenceConfig":{"maxTokens":100}}' \
  --cli-binary-format raw-in-base64-out \
  /tmp/bedrock-test.json && cat /tmp/bedrock-test.json
```

**If you get `ThrottlingException: Too many tokens per day`:**

You have hit your daily Bedrock token quota. This is common on new AWS accounts with default limits. Options:

1. Wait for the quota to reset (midnight UTC daily)
2. Request a quota increase: AWS Console > Service Quotas > Amazon Bedrock
3. Try a different model - first check what inference profiles are available in your region:

```bash
aws bedrock list-inference-profiles --region <region> \
  --query "inferenceProfileSummaries[].inferenceProfileId" \
  --output table
```

Note: in some regions like Tokyo (`ap-northeast-1`), bare model IDs are rejected. You must use inference profile IDs with a regional prefix (e.g. `apac.amazon.nova-micro-v1:0`).

If an alternative model works, update openclaw.json to use it:

```bash
python3 -c "
import json
with open('/home/ubuntu/.openclaw/openclaw.json', 'r') as f:
    config = json.load(f)
config['agents']['defaults']['model']['primary'] = 'amazon-bedrock/<new-model-id>'
with open('/home/ubuntu/.openclaw/openclaw.json', 'w') as f:
    json.dump(config, f, indent=2)
print('Done')
"
systemctl --user restart openclaw-gateway.service
```

**If you get `ResourceNotFoundException` for Claude models:**

Anthropic models on AWS require a one-time use case form per account. Go to AWS Console > Bedrock > Model access, find the Anthropic section, and fill out the form. It typically activates within 15 minutes.

---

### Port forwarding fails after reconnecting

**Symptom:** You previously had the dashboard working, closed the terminal, and now reconnecting gives `Connection to destination port failed`.

**Check if the gateway is still running:**

```bash
ss -tlnp | grep 18789
```

If nothing shows, the gateway crashed. Check the logs and restart:

```bash
journalctl --user -u openclaw-gateway.service -n 30 --no-pager
systemctl --user restart openclaw-gateway.service
sleep 15
ss -tlnp | grep 18789
```

Then kill and re-run the port forwarding command on your local machine.

---

### Do not click the update banner

OpenClaw shows an update available banner when a newer version exists. Do not click it. The CloudFormation template configures a specific version and the config format differs between versions. Updating in-place breaks the existing configuration. If you want to upgrade, redeploy the stack with the new version parameter instead.

---

## Things the Original Guide Doesn't Tell You

1. **Enable Bedrock model access first** - the deployment completes successfully but the AI silently fails to respond
2. **Disable the bonjour plugin immediately** - it will crash your gateway every 45 seconds in any cloud environment
3. **Don't click the update banner** - updating in-place can break the config the CloudFormation template set up
4. **Bedrock has daily token quotas on new accounts** - test Bedrock directly with `aws bedrock-runtime invoke-model` to confirm whether it is a quota issue
5. **In some regions (e.g. Tokyo), bare model IDs are rejected** - use inference profile IDs with a regional prefix and check with `aws bedrock list-inference-profiles`
6. **The gateway token lives in SSM Parameter Store** at `/openclaw/<stack-name>/gateway-token` - you do not need to save the dashboard URL separately
7. **Stopping the EC2 instance is safe** - config persists on the separate EBS data volume

---

## References

- [OpenClaw Documentation](https://docs.openclaw.ai)
- [aws-samples/sample-OpenClaw-on-AWS-with-Bedrock](https://github.com/aws-samples/sample-OpenClaw-on-AWS-with-Bedrock)
- [Original Guide - Jon Bonso / The Cloud Dojo](https://www.linkedin.com/pulse/stop-running-openclaw-your-laptop-how-i-built-secure-personal-bonso-thyyc/)
- [Amazon Bedrock Documentation](https://docs.aws.amazon.com/bedrock/)
- [SSM Session Manager Plugin Installation](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html)
- [Bedrock Inference Profiles](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles.html)
