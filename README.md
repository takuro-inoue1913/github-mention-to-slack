# github-mention-to-slack

自分が参加している GitHub Issue への他人の投稿を、5 分おきに Slack の Incoming Webhook へ転送する GitHub Actions。

## 通知対象

- GitHub 通知 API（`GET /notifications?participating=true`）に出る Issue のスレッド
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
  - schedule 実行は GitHub 側の混雑で数分〜十数分遅れることがある

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

## 動作確認

指定したコメントを 1 件だけ送る（前回チェック時刻は更新しない）。Issue / PR の会話タブのコメント（`#issuecomment-` の URL）のみ対応。

```sh
gh workflow run mention-to-slack -R takuro-inoue1913/github-mention-to-slack \
  -f comment_url='https://github.com/owner/repo/issues/1#issuecomment-123'
```

`comment_url` なしで実行すると通常のポーリングを 1 回行う。
