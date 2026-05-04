# LINE Messaging Integration

How to connect LINE to your OpenClaw instance running on AWS.

---

## Architecture

OpenClaw runs bound to loopback (`127.0.0.1:18789`) by design. LINE needs to send webhooks to a public HTTPS endpoint. So you need a public-facing layer to bridge the two.

```
LINE → Route53 (CNAME) → ALB (443 HTTPS) → EC2 nginx (80) → OpenClaw (127.0.0.1:18789)
```

**Do not change OpenClaw's `gateway.bind` to `0.0.0.0` to shortcut this.** That exposes port 18789 to the entire VPC, bypasses authentication, and there is a known cross-site WebSocket hijacking vulnerability (CVE-2026-25253) that gives attackers full control over an exposed OpenClaw instance. Keep it on loopback and use nginx as the proxy layer.

**AWS resources involved:**
- ALB in the public subnet (same VPC as the OpenClaw EC2 instance)
- ACM certificate for HTTPS termination on the ALB
- Route53 CNAME record pointing to the ALB DNS name
- ALB target group pointing to EC2 port 80
- EC2 security group allowing port 80 inbound from the ALB security group only
- nginx on the EC2 instance as the reverse proxy

> In this setup, the ALB, ACM certificate, and Route53 CNAME were provisioned by prompting OpenClaw to create the necessary AWS resources, which is a good demonstration of what it can actually do.

---

## Step 1: Confirm LINE is Available

LINE is a built-in integration in OpenClaw - it ships as part of the core package and does not require a separate plugin install. You can confirm it is present on your instance:

```bash
ls /usr/lib/node_modules/openclaw/dist/extensions/line/
```

If the directory exists, you are good to go. No install step needed.

---

## Step 2: Set Up LINE Developers Console

1. Go to [https://developers.line.biz/console/](https://developers.line.biz/console/)
2. Create a Provider if you don't already have one
3. Add a **Messaging API** channel under that provider
4. Go to the **Messaging API** tab of your new channel
5. Copy your **Channel access token** (generate one if needed)
6. Go to the **Basic settings** tab and copy your **Channel secret**
7. Back on the **Messaging API** tab, enable **Use webhook**
8. Set the **Webhook URL** to your domain:

```
https://your-domain.example.com/line/webhook
```

Keep this tab open - you'll need the token and secret in Step 5.

---

## Step 3: Install and Configure nginx

SSH or SSM into the EC2 instance.

**Install nginx:**

```bash
sudo apt-get install -y nginx
```

**Create the nginx config:**

```bash
sudo nano /etc/nginx/sites-available/openclaw
```

Paste this:

```nginx
server {
    listen 80;

    # Only proxy the LINE webhook path to OpenClaw
    location /line/webhook {
        proxy_pass http://127.0.0.1:18789;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_read_timeout 60s;
        client_max_body_size 10m;
    }

    # ALB health check endpoint
    location /health-messaging-integration {
        return 200 'ok';
        add_header Content-Type text/plain;
    }

    # Block everything else
    location / {
        return 403;
    }
}
```

**Enable the config:**

```bash
sudo ln -s /etc/nginx/sites-available/openclaw /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl enable nginx
sudo systemctl start nginx
```

**Verify it's working locally:**

```bash
curl -v http://127.0.0.1/health-messaging-integration
# Expected: HTTP/1.1 200 OK
```

---

## Step 4: Update AWS Resources

**EC2 Security Group:**
- Add inbound rule: port 80, source = ALB security group ID
- Remove any existing rule that allowed port 18789 from anywhere

**ALB Target Group:**
- Change the port from 18789 to 80
- Set health check path to `/health-messaging-integration`
- Wait for the target to show **Healthy** before proceeding

---

## Step 5: Configure OpenClaw for LINE

Edit `/home/ubuntu/.openclaw/openclaw.json` and add the `channels` block:

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "amazon-bedrock/global.anthropic.claude-sonnet-4-6"
      }
    }
  },
  "gateway": {
    "port": 18789,
    "mode": "local"
  },
  "channels": {
    "line": {
      "enabled": true,
      "channelAccessToken": "YOUR_CHANNEL_ACCESS_TOKEN",
      "channelSecret": "YOUR_CHANNEL_SECRET",
      "dmPolicy": "pairing"
    }
  }
}
```

Replace `YOUR_CHANNEL_ACCESS_TOKEN` and `YOUR_CHANNEL_SECRET` with the values from Step 2.

`dmPolicy: "pairing"` means new users must be approved before the bot will respond to them. Keep this setting - it controls who can interact with your bot.

After saving, restart the gateway:

```bash
systemctl --user restart openclaw-gateway.service
sleep 5
systemctl --user status openclaw-gateway.service
```

You should see a log line like:

```
[line] starting LINE provider (your-bot-name)
```

---

## Step 6: Verify the Webhook

Go back to the LINE Developers Console and click **Verify** on the webhook URL. It should return a success response.

If it returns an error, check:

1. nginx is running: `sudo systemctl status nginx`
2. ALB target group shows the EC2 instance as **Healthy**
3. Route53 CNAME is resolving to the ALB: `nslookup your-domain.example.com`
4. ALB listener is set to port 443 with your ACM certificate
5. OpenClaw is running and the LINE provider started: `journalctl --user -u openclaw-gateway.service -n 30 --no-pager`

---

## Step 7: Pair Users

Because `dmPolicy` is set to `pairing`, the bot will not respond to anyone until you explicitly approve them.

**To add a user:**
1. Have the user send a message to your LINE bot
2. They will receive a pairing code (e.g. `CE6AVY9N`)
3. SSM or SSH into the EC2 instance and run:

```bash
openclaw pairing approve line CE6AVY9N
```

4. The user can now interact with the bot

**To see pending pairing requests:**

```bash
openclaw pairing list
```

Keep this list tight. The bot has access to AWS actions through the IAM role attached to the EC2 instance. Only approve users who are meant to have that access.

---

## Troubleshooting

**ALB returns 502**

OpenClaw is still binding to loopback and nginx is not running or not configured yet. Complete Step 3 first and verify the nginx health check returns 200.

**LINE webhook verification fails with 403**

nginx is running but the webhook path is wrong. The config should proxy `/line/webhook` specifically. The `location /` block returns 403 for everything else, which is correct - just make sure the Webhook URL in LINE Developers Console ends with `/line/webhook`.

**LINE webhook verification succeeds but the bot does not respond**

The OpenClaw LINE provider did not start. Check the gateway logs:

```bash
journalctl --user -u openclaw-gateway.service -n 50 --no-pager
```

Look for `[line] starting LINE provider`. If it is not there, the channel config in `openclaw.json` is missing or has a typo. Also make sure the gateway restarted after you edited the config.

**Bot responds but only with errors**

Check the model config and Bedrock access. Confirm the Bedrock credentials in `/home/ubuntu/.openclaw/aws-credentials.env` are correct and the model ID is available in your region.

**Gateway keeps restarting while you are setting up the LINE channel**

This is expected. Every time you save `openclaw.json`, OpenClaw hot-reloads via SIGUSR1. It is not a problem - once you stop editing the config, the restarts stop and the connection stabilizes.
