# OpenClaw on AWS with Amazon Bedrock

A self-hosted personal AI assistant running on AWS EC2, powered by Amazon Bedrock. No separate API keys to manage, no laptop dependency, runs 24/7.

Based on the original guide by [Jon Bonso / The Cloud Dojo](https://www.linkedin.com/pulse/stop-running-openclaw-your-laptop-how-i-built-secure-personal-bonso-thyyc/).

---

## What This Is

[OpenClaw](https://openclaw.ai) is an open-source AI agent framework. Unlike a simple chatbot, it can take actions - respond to messages on WhatsApp, Telegram, or Discord; run scheduled tasks; maintain memory across conversations; and use tools and skills.

The problem with running it locally: it needs to be on 24/7, has full access to your machine, and dies when your laptop sleeps or your Wi-Fi drops.

This deployment moves it to AWS. It runs continuously on EC2 and calls Amazon Bedrock for the AI model, so you get a single unified API instead of juggling separate keys for each provider.

---

## Templates

| Template | Description |
|---|---|
| [`cloudformation/openclaw-bedrock_v1_2.yaml`](cloudformation/openclaw-bedrock_v1_2.yaml) | EC2 in private subnet, NAT Gateway, SSM access, Bedrock via EC2 instance role |
| [`cloudformation/openclaw-bedrock_v1_3.yaml`](cloudformation/openclaw-bedrock_v1_3.yaml) | v1.2 + dedicated IAM user for Bedrock, credentials in SSM Parameter Store, automated UserData bootstrap |

Use v1.3 for new deployments. v1.2 is kept here for reference.

---

## Architecture

```
Your Phone (Line / Telegram / Discord / Slack)
            |
            v
    Messaging Platform
            |
            v
  OpenClaw Gateway (EC2 - Ubuntu 24.04, Graviton ARM64)
  Private Subnet, no public IP
  Access via SSM Session Manager only
            |
            v
  Amazon Bedrock (Claude / Nova / Llama / DeepSeek)
```

**Components:**

- **EC2 instance** - runs the OpenClaw gateway process (t4g.small by default, ARM64/Graviton)
- **Amazon Bedrock** - provides the AI model via a single unified AWS API
- **Dedicated IAM user (v1.3)** - scoped to Bedrock invoke only; keys stored in SSM, injected into the OpenClaw process at boot
- **EBS data volume** - 30GB encrypted gp3, separate from the root volume; persists config across instance stop/start
- **NAT Gateway** - outbound internet for the private subnet (needed at boot to download Node.js and OpenClaw)
- **VPC endpoints** (optional) - private connectivity to Bedrock, SSM, and EC2 Messages so traffic never leaves AWS
- **SSM Session Manager** - primary access method; no SSH port open, no key pair required for day-to-day use

---

## Why v1.3 Uses a Dedicated IAM User (and Not the Instance Profile)

This is the non-obvious design decision, so it's worth documenting.

The expected AWS-native pattern would be: attach an IAM role to the EC2 instance, let the instance profile deliver temporary credentials automatically via the metadata service (`169.254.169.254`). No static keys, no SSM, clean.

The problem is OpenClaw runs on a library called `pi-coding-agent`, and **that library does not talk to the EC2 metadata service**. It only reads credentials from explicit environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`). So even though the instance profile exists and the AWS CLI on the instance can use it fine, OpenClaw itself cannot.

The workaround in v1.3:
1. Create a dedicated IAM user scoped only to `bedrock:InvokeModel` and related actions
2. CloudFormation generates the access key and stores it in SSM Parameter Store
3. At boot, the UserData script uses the AWS CLI (which CAN use the metadata service) to fetch the keys from SSM
4. The keys are written to `/home/ubuntu/.openclaw/aws-credentials.env` (loaded by systemd as an EnvironmentFile) and to OpenClaw's own `auth-profiles.json`
5. The OpenClaw process reads the env vars directly - no metadata service needed

Static IAM user keys do not expire, which is why no credential refresh mechanism is needed. The tradeoff is that static keys are less secure than temporary STS credentials. For a personal sandbox deployment this is acceptable; for a production environment, consider Secrets Manager and a credential rotation strategy.

If you want to use IAM roles instead of a static IAM user, it's possible but requires a refresh script, because STS-issued temporary credentials expire in at most 12 hours and OpenClaw does not re-read them on its own.

---

## Prerequisites

Before deploying, complete these steps in order. The deployment will succeed even if you skip some of them, but OpenClaw will silently fail to work.

### 1. Enable Bedrock Model Access

This is the step most people miss. Your AWS account does not have access to Bedrock models by default - you have to explicitly request it.

1. Open the [Amazon Bedrock console](https://console.aws.amazon.com/bedrock)
2. Go to **Model access** in the left sidebar
3. Find the model you plan to use (e.g. Amazon Nova, Anthropic Claude, Meta Llama)
4. Click **Manage model access** and enable it
5. For Anthropic models specifically, you need to fill out a short use case form - approval is usually instant but required

Do this before running the CloudFormation stack. If you skip it, the stack will deploy successfully, the gateway will start, and you will get no response when you try to chat. There is no error message - it just does nothing.

### 2. Install the SSM Session Manager Plugin

You need this on your local machine to run port forwarding. Without it, you cannot access the OpenClaw dashboard in your browser.

Follow the [AWS installation guide](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html) for your OS.

### 3. Create an EC2 Key Pair (Optional but Recommended)

The template uses SSM Session Manager as the primary access method, so a key pair is not strictly required. But you should have one if you ever need emergency SSH access or want to troubleshoot outside of SSM.

To create one:

1. Go to the [EC2 console](https://console.aws.amazon.com/ec2) in your target region
2. Navigate to **Network & Security > Key Pairs**
3. Click **Create key pair**
4. Give it a name (e.g. `openclaw-key`), choose RSA, format `.pem`
5. Download and store the `.pem` file somewhere safe (e.g. `~/.ssh/`)
6. Run `chmod 400 ~/.ssh/openclaw-key.pem`

When deploying the CloudFormation stack, enter this key pair name in the `KeyPairName` parameter. The key pair name is case-sensitive - if it doesn't match exactly what you typed in the EC2 console, CloudFormation will fail.

### 4. Choose Your Region and Model

Not all Bedrock models are available in all regions. For regions outside `us-east-1` and `us-west-2`, bare model IDs are often rejected - you need to use inference profile IDs with a regional prefix.

Check what's available in your target region:

```bash
aws bedrock list-inference-profiles --region YOUR_REGION
```

Examples of correct inference profile IDs by region:

| Region | Profile format |
|---|---|
| us-east-1 / us-west-2 | `us.amazon.nova-lite-v1:0` |
| ap-northeast-1 (Tokyo) | `ap.amazon.nova-lite-v1:0` |
| eu-central-1 | `eu.amazon.nova-lite-v1:0` |
| Global (cross-region) | `global.amazon.nova-2-lite-v1:0` |

If you select a model that your region doesn't support, the deployment will succeed but the AI will not respond.

---

## Deployment

### Deploy the Stack

1. Download [`cloudformation/openclaw-bedrock_v1_3.yaml`](cloudformation/openclaw-bedrock_v1_3.yaml)

2. Open the [CloudFormation console](https://console.aws.amazon.com/cloudformation) in your target region

3. Click **Create stack > With new resources**

4. Upload the template file

5. Fill in parameters:

| Parameter | Recommended setting |
|---|---|
| `OpenClawModel` | Choose a model available in your region (see above) |
| `OpenClawVersion` | `2026.3.24` - stable and tested; do not use `latest` |
| `InstanceType` | `t4g.small` for personal use; Graviton (ARM) is 20-40% cheaper |
| `KeyPairName` | Your key pair name, or leave `none` if not using SSH |
| `CreateVPCEndpoints` | `false` for sandbox (saves ~$29/month); `true` for production |
| `EnableDataProtection` | `false` for sandbox; `true` if you want to retain data on stack delete |

6. Click through to **Create stack**

7. Wait for the stack to reach `CREATE_COMPLETE` - this takes 10-20 minutes because CloudFormation waits for the OpenClaw gateway to finish starting up

### Connect to the Dashboard

Once the stack is complete, go to the **Outputs** tab and follow the steps there:

**Step 1:** Install the SSM Session Manager plugin (if you haven't already)

**Step 2:** Run the port forwarding command from the Outputs. It looks like this:

```bash
aws ssm start-session \
  --target i-0abc123def456789 \
  --region ap-northeast-1 \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["18789"],"localPortNumber":["18789"]}'
```

Keep this terminal open. It's the tunnel.

**Step 3:** Open the URL from the Outputs in your browser:

```
http://localhost:18789/?token=<your-token>
```

**Step 4:** Connect a messaging channel (WhatsApp, Line, Telegram, Discord, Slack) via the dashboard.

---

## Estimated Monthly Cost

| Resource | Cost |
|---|---|
| EC2 t4g.small | ~$12-15 |
| EBS 30GB gp3 | ~$2.40 |
| NAT Gateway | ~$32-45 |
| VPC Endpoints (5 endpoints, if enabled) | ~$29 |
| Bedrock | Pay per token |
| **Total without endpoints** | **~$46-62/month** |
| **Total with endpoints** | **~$75-90/month** |

The NAT Gateway is the biggest fixed cost. It's needed at boot for the instance to download Node.js and OpenClaw from the internet. After setup, you could theoretically remove it if you only use VPC endpoints - but this makes the stack more complex to manage.

To save cost on a sandbox deployment: set `CreateVPCEndpoints=false` and use `t4g.small`.

---

## Things the Original Guide Does Not Tell You

These are hard-won findings from deploying and debugging this setup from scratch.

**1. Enable Bedrock model access before you deploy**

If you skip this, the stack creates fine, OpenClaw starts, and there is no error message - the AI just does not respond. Go to Bedrock > Model access and enable it first.

**2. Disable the bonjour plugin immediately after setup**

The bonjour plugin crashes the OpenClaw gateway every ~45 seconds in any cloud environment. The symptom is the gateway repeatedly dying and restarting.

Check if this is happening:

```bash
journalctl --user -u openclaw-gateway.service -n 50 --no-pager
```

If you see repeated restarts with socket/network errors, bonjour is the cause. Disable it:

```bash
# SSH into the instance or use SSM
cat ~/.openclaw/openclaw.json
```

The config needs this section:

```json
"plugins": {
  "entries": {
    "bonjour": {
      "enabled": false
    }
  }
}
```

Then restart the service:

```bash
systemctl --user restart openclaw-gateway.service
```

**3. Do not click the update banner in the OpenClaw dashboard**

In-place updates can overwrite or break the config file the CloudFormation template set up. If you want to update, redeploy the stack with a new version.

**4. Bedrock has daily token quotas on new accounts**

If the AI stops responding and there are no obvious errors, check your Bedrock quota in the AWS console under Service Quotas > Amazon Bedrock. You can also test Bedrock directly from the instance to isolate the issue:

```bash
aws bedrock-runtime invoke-model \
  --model-id amazon.nova-lite-v1:0 \
  --body '{"messages":[{"role":"user","content":[{"type":"text","text":"hello"}]}],"inferenceConfig":{"max_new_tokens":100}}' \
  --cli-binary-format raw-in-base64-out \
  /tmp/bedrock-test.json && cat /tmp/bedrock-test.json
```

If this returns a response, Bedrock is working and the issue is in OpenClaw's config or credential setup.

**5. Some regions reject bare model IDs - use inference profile IDs**

In regions like Tokyo (`ap-northeast-1`), a model ID like `amazon.nova-lite-v1:0` gets rejected. You must use the regional inference profile like `ap.amazon.nova-lite-v1:0`.

To list what's available in your region:

```bash
aws bedrock list-inference-profiles --region YOUR_REGION
```

**6. Key pair name is case-sensitive**

If you enter a key pair name that doesn't exactly match what you created in EC2 (including capitalization), CloudFormation will fail at the EC2 instance resource. Double-check the exact name in the EC2 console.

**7. The gateway token is in SSM - you do not need to save the URL**

The access token for the dashboard is stored in SSM Parameter Store at:

```
/openclaw/<stack-name>/gateway-token
```

To retrieve it if you lose the URL:

```bash
aws ssm get-parameter \
  --name /openclaw/YOUR_STACK_NAME/gateway-token \
  --with-decryption \
  --query Parameter.Value \
  --output text \
  --region YOUR_REGION
```

**8. Stopping the EC2 instance is safe**

Config and data live on the separate 30GB EBS volume (`/dev/sdf`), not on the root volume. You can stop the instance to save cost when not using it. The gateway token persists in SSM as well.

---

## Troubleshooting

### Stack creation fails or times out

The CloudFormation WaitCondition gives the setup script 20 minutes to complete. If it times out, the most likely cause is one of:

- Node.js or OpenClaw failed to download (transient network issue - try redeploying)
- The OpenClaw gateway didn't start within 4 minutes

To see exactly what happened:

```bash
# SSM into the instance after the stack fails
aws ssm start-session --target INSTANCE_ID --region YOUR_REGION

# Then on the instance:
sudo tail -100 /var/log/openclaw-setup.log
```

### Gateway crashes in a loop

```bash
journalctl --user -u openclaw-gateway.service -n 100 --no-pager
```

Look for repeated restart entries. Most commonly caused by the bonjour plugin (see the gotchas section above).

### OpenClaw responds but Bedrock calls fail

```bash
# Check the credentials env file was written correctly
cat /home/ubuntu/.openclaw/aws-credentials.env

# Test Bedrock directly with the AWS CLI
aws bedrock-runtime invoke-model \
  --model-id YOUR_MODEL_ID \
  --region YOUR_REGION \
  --body '{"messages":[{"role":"user","content":[{"type":"text","text":"test"}]}],"inferenceConfig":{"max_new_tokens":50}}' \
  --cli-binary-format raw-in-base64-out \
  /tmp/test-out.json && cat /tmp/test-out.json
```

If the AWS CLI call works but OpenClaw doesn't, the issue is in how credentials are being passed to the OpenClaw process. Check the env file and `auth-profiles.json`.

### Port forwarding disconnects

The SSM session will time out after 20 minutes of inactivity by default. Just re-run the port forwarding command from the Outputs tab.


## Demo
[Watch Demo: Chatting with OpenClaw via Line](https://1drv.ms/v/c/060d23632df8ec38/IQC0GsULKl5vQbGlm-2_u-9PAfIvJcA52rRzNVS4MX-VOwA?e=62fRyP)

---

## Version History

**v1.3 (current)**
- Dedicated IAM user created for Bedrock access, scoped to `InvokeModel` only
- Access key and secret stored in SSM Parameter Store
- UserData fetches credentials from SSM at boot and writes them to the systemd EnvironmentFile and OpenClaw's `auth-profiles.json`
- `AWS_REGION` added to systemd service environment
- Default model changed to `global.anthropic.claude-sonnet-4-6`
- EC2 instance now depends on SSM parameters being written before boot

**v1.2**
- EC2 instance moved to private subnet (no public IP)
- NAT Gateway added to public subnet for outbound internet during setup
- Private route table with default route via NAT Gateway
- Access via SSM Session Manager only (no SSH required)
- Bedrock access via EC2 instance role directly

---

## References

- [OpenClaw Documentation](https://docs.openclaw.ai)
- [aws-samples/sample-OpenClaw-on-AWS-with-Bedrock](https://github.com/aws-samples/sample-OpenClaw-on-AWS-with-Bedrock)
- [Original Guide - Jon Bonso / The Cloud Dojo](https://www.linkedin.com/pulse/stop-running-openclaw-your-laptop-how-i-built-secure-personal-bonso-thyyc/)
- [Amazon Bedrock Documentation](https://docs.aws.amazon.com/bedrock/)
- [SSM Session Manager Plugin Installation](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html)
- [Bedrock Inference Profiles](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles.html)
