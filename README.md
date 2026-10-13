# github-mention-to-slack

自分が参加している GitHub Issue への他人の投稿を、1 分おきに Slack の Incoming Webhook へ転送する GitHub Actions。

## 通知対象

- GitHub 通知 API（`GET /notifications?participating=true`）に出る Issue のスレッドのうち、`hello-ai` Org のリポジトリのもの
  - 対象 Org は workflow の `TARGET_OWNER` で指定する
  - 自分が作成した・コメントした・アサインされた・メンションされた Issue が該当する
- そのスレッドで前回チェック以降に投稿された Issue 本文（新規作成時）とコメント
- 対象外
  - 自分の投稿
  - PR（GitHub の Scheduled reminders の real-time alerts で受け取る前提）
  - PR の行コメント、編集で後から追加された内容

## メッセージ形式

```
参加しているIssueで *投稿者* のコメントが投稿されました   ← Issue 作成時は「がIssueを作成しました」
<コメントへのリンク|owner/repo#番号 Issue タイトル>
> 本文（空行を詰めて 5 行・300 文字まで。超えたら末尾に「 …」）
```

- 本文中の `@自分の GitHub ユーザー名` は Slack メンション（`<@SLACK_MEMBER_ID>`）に変換する（大文字小文字は区別しない）
- `&` `<` `>` は Slack 用にエスケープする

## 重複・取りこぼしの扱い

- 前回チェックした時刻を Actions cache（`last_checked`）に保存し、それ以降に作成された投稿だけを送る
  - 初回（cache なし）は直近 10 分を対象にする
- notify job の `concurrency` で同時実行を防ぎ、遅延した実行同士で同じ投稿を送らないようにしている
- 限界
  - 途中で Slack への送信が失敗すると時刻が更新されず、次回にそれまで送った分も再送される
  - schedule 実行は GitHub 側で間引かれ、1 日数回しか動かないことがある（外部 cron からの起動で補う）

## 起動方法

GitHub の schedule は混雑時に間引かれ、`*/5` でも数時間空くことがある。そのため外部 cron（cron-job.org）から 1 分おきに `workflow_dispatch` を呼んで起動する。

- 通知 API のポーリング間隔（`X-Poll-Interval`）が 60 秒なので、1 分より短くしても意味がない
- 実行が重なっても `concurrency` で直列化され、待機中の古い実行はキャンセルされる
- `*/5` の schedule は外部 cron が止まったときの予備として残している
- Webhook でのリアルタイム化は不可。参加中 Issue の通知に対応する Webhook がなく、他 Org のリポジトリには Webhook を設定できない

## schedule の自動停止対策

Public リポジトリは 60 日間動きがないと schedule が自動停止する。毎月 1 日 0:00（UTC）に `keepalive` job が workflow の有効化 API（`PUT /repos/{owner}/{repo}/actions/workflows/{id}/enable`）を呼び、非アクティブ期間をリセットする。止まってしまった場合は Actions タブから再有効化する。

## セットアップ

### GitHub の通知設定

https://github.com/settings/notifications の「Participating, @mentions and custom」で **On GitHub** を有効にする。無効だと通知 API にスレッドが出ず、何も送られない。

### Secrets / Variables

| 種類 | 名前 | 内容 |
|---|---|---|
| Secret | `NOTIFICATIONS_TOKEN` | Classic PAT（scope: `notifications`, `repo`）。SAML SSO の Org は「Configure SSO」で Authorize が必要 |
| Secret | `SLACK_WEBHOOK_URL` | Slack Incoming Webhook の URL。送り先を変えるときは Webhook を作り直してここを差し替える |
| Variable | `SLACK_MEMBER_ID` | 自分の Slack メンバー ID（メンション変換に使う） |

### 外部 cron（cron-job.org）

1. Fine-grained PAT を作る
   - Repository access: `takuro-inoue1913/github-mention-to-slack` のみ
   - Permissions: Actions を **Read and write**
   - 有効期限を設定した場合は、切れる前に作り直して cron-job.org の Header を差し替える
2. https://cron-job.org で cronjob を作る
   - URL: `https://api.github.com/repos/takuro-inoue1913/github-mention-to-slack/actions/workflows/mention-to-slack.yml/dispatches`
   - Schedule: Every 1 minute
   - Advanced
     - Request method: `POST`
     - Headers
       - `Authorization`: `Bearer <PAT>`
       - `Accept`: `application/vnd.github+json`
       - `X-GitHub-Api-Version`: `2022-11-28`
     - Request body: `{"ref":"main"}`
3. 「Test run」で 2xx が返り、Actions タブに `workflow_dispatch` の実行が出ることを確認する

## 動作確認

指定したコメントを 1 件だけ送る（前回チェック時刻は更新しない）。Issue / PR の会話タブのコメント（`#issuecomment-` の URL）のみ対応。

```sh
gh workflow run mention-to-slack -R takuro-inoue1913/github-mention-to-slack \
  -f comment_url='https://github.com/owner/repo/issues/1#issuecomment-123'
```

`comment_url` なしで実行すると通常のポーリングを 1 回行う。
