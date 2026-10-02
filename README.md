# github-mention-to-slack

GitHub の通知 API を 5 分おきにポーリングし、自分宛てのメンション（Issue / PR）を Slack の Incoming Webhook に転送する。

## Secrets

- `NOTIFICATIONS_TOKEN`: Classic PAT（scope: `notifications`, `repo`。SAML SSO の Org は Authorize が必要）
- `SLACK_WEBHOOK_URL`: Slack Incoming Webhook の URL
