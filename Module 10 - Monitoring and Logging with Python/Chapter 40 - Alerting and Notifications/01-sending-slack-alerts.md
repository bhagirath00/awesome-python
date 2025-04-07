# Sending Slack Alerts: Rich Formatting and Rate Limit Handling

When a system check fails or a metric exceeds its threshold, your automation script must immediately notify the engineering team. **Slack** is a very popular destination for these alerts.

However, sending flat text alerts like "server down" is not very helpful. An actionable alert should highlight severity, display metadata (such as server IP and error codes) in clean columns, and include links to troubleshooting playbooks.

In this guide, we will connect to Slack Incoming Webhooks, design rich alert cards using **Block Kit**, and implement handling logic to manage Slack's **rate limit (HTTP 429)** blocks.

---

## Slack Webhook Configuration

1. In your Slack workspace, create a dedicated channel (e.g., `#alerts-devops`).
2. Search for the **Incoming Webhooks** app in the Slack App Directory and add it.
3. Configure the webhook to post to your alerts channel.
4. Copy the generated Webhook URL.

---

## 1. Designing Alerts with Block Kit

Slack provides a layout framework called **Block Kit**. Instead of parsing raw markdown text, Block Kit uses JSON block elements to format content.

Supported blocks include:
* **`header`**: Large, bold header text.
* **`section`**: Standard text layout blocks.
* **`fields`**: Key-value pairs displayed in columns.
* **`context`**: Small footnote text (useful for source details).
* **`actions`**: Interactive buttons.

Here is a function that formats and sends a multi-block, color-coded Slack alert:

```python
import os
import requests

def send_rich_slack_alert(title, severity, description, metadata_dict, playbook_url=None):
    webhook_url = os.environ.get("SLACK_WEBHOOK_URL")
    if not webhook_url:
        print("Error: SLACK_WEBHOOK_URL environment variable is not set.")
        return False

    # Choose color and emoji based on severity
    severity_configs = {
        "INFO": {"color": "#36a64f", "emoji": "ℹ️"},
        "WARNING": {"color": "#ff9900", "emoji": "⚠️"},
        "CRITICAL": {"color": "#ff0000", "emoji": "🚨"}
    }
    config = severity_configs.get(severity.upper(), {"color": "#808080", "emoji": "🔔"})

    # 1. Format metadata fields into columns
    fields = []
    for key, value in metadata_dict.items():
        fields.append({"type": "mrkdwn", "text": f"*{key}:*\n{value}"})

    # 2. Build Block Kit structure
    blocks = [
        {
            "type": "section",
            "text": {
                "type": "mrkdwn",
                "text": f"{config['emoji']} *[{severity.upper()}] {title}*"
            }
        },
        {
            "type": "section",
            "text": {
                "type": "mrkdwn",
                "text": description
            }
        }
    ]

    # Append metadata fields if they exist
    if fields:
        blocks.append({"type": "section", "fields": fields})

    # Append interactive action buttons if a playbook is provided
    if playbook_url:
        blocks.append({
            "type": "actions",
            "elements": [
                {
                    "type": "button",
                    "text": {"type": "plain_text", "text": "📖 View Playbook"},
                    "url": playbook_url,
                    "style": "primary"
                }
            ]
        })

    # 3. Wrap blocks inside attachments to enable the colored side bar
    payload = {
        "attachments": [
            {
                "color": config["color"],
                "blocks": blocks
            }
        ]
    }

    try:
        response = requests.post(webhook_url, json=payload, timeout=5)
        return response.status_code == 200
    except requests.RequestException as e:
        print(f"Failed to send Slack alert: {e}")
        return False
```

---

## 2. Managing Slack's Rate Limits (HTTP 429)

Slack Incoming Webhooks enforce a rate limit of **1 request per second**. If your monitoring script encounters an outage and tries to send hundreds of alerts at once, Slack will block the requests and return an **HTTP 429 Too Many Requests** status code.

To handle this, your code must inspect the status code, read the `Retry-After` header (which indicates how many seconds to wait), and retry after a delay.

```python
import time
import requests

def send_slack_with_rate_limit_handling(webhook_url, payload, max_retries=3):
    for attempt in range(1, max_retries + 1):
        try:
            response = requests.post(webhook_url, json=payload, timeout=5)
            
            # Check for success
            if response.status_code == 200:
                return True
                
            # Handle rate limit
            elif response.status_code == 429:
                # Read the wait time from headers (default to 5s if missing)
                retry_after = int(response.headers.get("Retry-After", 5))
                print(f"⚠️ Slack Rate Limit hit! Waiting {retry_after} seconds before retrying...")
                time.sleep(retry_after)
            else:
                print(f"❌ Slack API returned error: {response.status_code} - {response.text}")
                return False
                
        except requests.RequestException as e:
            print(f"Connection failure on attempt {attempt}: {e}")
            time.sleep(2)
            
    print("❌ Failed to send Slack alert after retries.")
    return False
```

---

## Summary

Slack alerts keep your engineering team aligned in real time. By designing formatted Block Kit cards and implementing rate limit handling logic, you can deliver actionable alerts securely. In the next section, we will cover **Email alerts** via SMTP.
