# github-mention-to-slack

GitHub の通知 API を 5 分おきにポーリングし、自分が作成・コメント・アサイン・メンションされた Issue への他人の投稿を Slack の Incoming Webhook に転送する。

## Secrets

- `NOTIFICATIONS_TOKEN`: Classic PAT（scope: `notifications`, `repo`。SAML SSO の Org は Authorize が必要）
- `SLACK_WEBHOOK_URL`: Slack Incoming Webhook の URL
