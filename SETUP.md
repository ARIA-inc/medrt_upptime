# MEDRT Upptime セットアップ手順

本番環境（VPS 上の docker-compose）の死活監視を、GitHub Actions のみで行う構成です。
サーバー不要・追加コスト無しで、5分間隔の外形監視 → 障害検知 → GitHub Issue 自動起票 → Slack 通知まで行います。

---

## 1. 監視対象

`.upptimerc.yml` の `sites` で定義します。

| name | URL | 備考 |
| --- | --- | --- |
| `medrt_web_prd` | https://vn.job.medrthub.com/ | 医師サイト（本番 / VPS） |
| `medrt_api_prd` | https://api.vn.job.medrthub.com/health | API ヘルスチェック（200 / `OK` を返却） |
| `medrt_management_prd` | https://admin.vn.job.medrthub.com/ | 管理サイト（本番 / VPS） |
| `medrt_institution_management_prd` | https://organization.vn.job.medrthub.com/ | 医療機関管理サイト（本番 / VPS） |

ステージング（AWS ECS/RDS）も同ファイル内にコメントで用意済みです。
ただし STG は `medrt_cdk` の `monitoring-stack.ts` で CloudWatch + SNS による監視が既に有効なため、
重複が不要であれば無効のままで問題ありません。

---

## 2. リポジトリ作成と push

```bash
cd medrt_upptime

# GitHub 上にリポジトリを作成（Public 推奨。理由は「4. GitHub Pages」を参照）
gh repo create ARIA-inc/medrt_upptime --public --source=. --remote=origin --push
```

> リポジトリ名を `medrt_upptime` 以外にする場合は、`.upptimerc.yml` の `repo:` と
> `status-website.baseUrl` も同じ名前に変更してください。

---

## 3. GitHub 側の設定

### 3-1. Actions の書き込み権限

`Settings → Actions → General → Workflow permissions`
→ **Read and write permissions** を選択して Save。

Upptime は監視結果を `history/` `api/` `graphs/` にコミットするため、書き込み権限が必須です。

### 3-2. GH_PAT（推奨）

`github.token` でも動作しますが、ワークフローから別ワークフローを起動する処理
（Graphs CI の dispatch など）が動かないため、Personal Access Token の登録を推奨します。

1. https://github.com/settings/tokens → **Generate new token (classic)**
2. スコープ: `repo` と `workflow` にチェック（Public リポジトリのみなら `public_repo` でも可）
3. 有効期限は無期限または長期（期限切れで監視が止まるため）
4. `Settings → Secrets and variables → Actions → New repository secret`
   - Name: `GH_PAT`
   - Value: 発行したトークン

> ARIA-inc 組織で SSO が有効な場合、トークン発行後に **Configure SSO → Authorize** を実行してください。

---

## 4. GitHub Pages（ステータスページ）

`Static Site CI` が `gh-pages` ブランチへ静的サイトを push します（`peaceiris/actions-gh-pages`）。

1. 初回は `Actions → Static Site CI → Run workflow` を手動実行（`gh-pages` ブランチが生成される）
2. `Settings → Pages`
   - Source: **Deploy from a branch**
   - Branch: **gh-pages** / **/ (root)**
3. 数分後に https://ARIA-inc.github.io/medrt_upptime で公開されます

### Public / Private の判断

| | Public | Private |
| --- | --- | --- |
| 監視・Issue 起票・Slack 通知 | ✅ | ✅ |
| ステータスページ（GitHub Pages） | ✅ | ❌ Enterprise Cloud のみ |
| Actions 実行時間 | 無料・無制限 | 無料枠を消費（Free 2,000分/月・Team 3,000分/月） |

Private でも監視とアラートは問題なく動作します。Public が必要なのは **ステータスページを公開する場合のみ** です。

Actions の消費量は、`ema_sound_upptime` の実績値（1回あたり約12〜20秒、
GitHub の課金は1分未満切り上げ）から月あたり約1,200分と見積もられます。
無料枠には収まりますが余裕は大きくないため、Private 運用時は使用量を確認してください。

このリポジトリには機密情報は含まれません（Webhook URL や PAT は GitHub Secrets 側に保存され、
コードには入りません）。監視対象の URL は既に公開されているものです。
参考の `ema_sound_upptime` も Public です。

独自ドメイン（例 `status.medrthub.com`）を使う場合は、`.upptimerc.yml` の
`status-website.cname` を設定し、`baseUrl` の行を削除してください。

---

## 5. Slack 通知の設定

### 5-1. Slack 側：Incoming Webhook を発行

1. https://api.slack.com/apps → **Create New App** → **From scratch**
   - App Name: `MEDRT Uptime`（任意）
   - Workspace: 通知先のワークスペース
2. 左メニュー **Incoming Webhooks** → **Activate Incoming Webhooks** を **On**
3. **Add New Webhook to Workspace** → 通知先チャンネル（例 `#medrt-alert`）を選択 → **Allow**
4. 発行された Webhook URL をコピー
   （`https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX`）

> 既存の Slack App / Workflow Builder の Webhook を流用しても構いません。
> 投稿先チャンネルは Webhook URL 側で固定されます。

### 5-2. GitHub 側：シークレットを登録

`Settings → Secrets and variables → Actions → New repository secret` で 2 つ登録します。

| Secret 名 | 値 | 必須 |
| --- | --- | --- |
| `NOTIFICATION_SLACK` | `true` | ✅ |
| `NOTIFICATION_SLACK_WEBHOOK_URL` | 5-1 でコピーした Webhook URL | ✅ |

CLI の場合:

```bash
gh secret set NOTIFICATION_SLACK --body "true" --repo ARIA-inc/medrt_upptime
gh secret set NOTIFICATION_SLACK_WEBHOOK_URL --body "https://hooks.slack.com/services/XXX/YYY/ZZZ" --repo ARIA-inc/medrt_upptime
```

> `NOTIFICATION_SLACK` が `true` かつ Webhook URL が設定されている場合のみ Slack 通知が有効になります。
> 両方揃っていないと通知は送信されません（監視自体は動作します）。

### 5-3. 通知文言のカスタマイズ（任意）

未設定の場合は既定の英語メッセージが送信されます。

- ダウン時: `🟥 medrt_web_prd (https://vn.job.medrthub.com/) is **down** : <Issue URL>`
- 復旧時: `🟩 medrt_web_prd (https://vn.job.medrthub.com/) is back up`

| Secret 名 | 説明 | 利用できる変数 |
| --- | --- | --- |
| `NOTIFICATIONS_DOWN_MESSAGE` | ダウン／性能低下の検知時 | `$SITE_NAME` `$SITE_URL` `$ISSUE_URL` `$RESPONSE_CODE` `$STATUS` `$EMOJI` |
| `NOTIFICATIONS_UP_MESSAGE` | 復旧時 | `$SITE_NAME` `$SITE_URL` `$STATUS` `$EMOJI` |

例（日本語化）:

```bash
gh secret set NOTIFICATIONS_DOWN_MESSAGE \
  --body '🟥 【本番障害】$SITE_NAME $SITE_URL がダウンしました（HTTP $RESPONSE_CODE）詳細: $ISSUE_URL' \
  --repo ARIA-inc/medrt_upptime
gh secret set NOTIFICATIONS_UP_MESSAGE \
  --body '🟩 【復旧】$SITE_NAME $SITE_URL が復旧しました' \
  --repo ARIA-inc/medrt_upptime
```

> 注意点:
> - `$SITE_URL` は括弧付き（`(https://…)`）に置換されるため、文中で自分で括弧を付ける必要はありません。
> - 各変数の置換は先頭 1 箇所のみです（同じ変数を 2 回使わないでください）。
> - 性能低下（degraded）時も `NOTIFICATIONS_DOWN_MESSAGE` が使われ、`$EMOJI` は 🟨 になります。

### 5-4. メール通知を併用する場合（任意）

SMTP / SendGrid / SES などが利用できます。SMTP の例:

`NOTIFICATION_EMAIL_SMTP=true`, `NOTIFICATION_EMAIL_SMTP_HOST`, `NOTIFICATION_EMAIL_SMTP_PORT`,
`NOTIFICATION_EMAIL_SMTP_USERNAME`, `NOTIFICATION_EMAIL_SMTP_PASSWORD`,
`NOTIFICATION_EMAIL_FROM`, `NOTIFICATION_EMAIL_TO`

詳細: https://upptime.js.org/docs/notifications

---

## 6. 動作確認

### 6-0. ワークフローが登録されているか確認（初回のみ・重要）

```bash
gh api /repos/ARIA-inc/medrt_upptime/actions/workflows --jq '.total_count'
```

**9** が返れば正常です。少ない場合、GitHub が初回 push を Actions として処理せず、
schedule / workflow_dispatch のみのワークフロー（Uptime CI など）が未登録の状態です。
この状態では `gh workflow run "Uptime CI"` が
`HTTP 404: workflow uptime.yml not found on the default branch` になり、
`Setup CI` のログにも `Unable to find workflow 'Graphs CI'` が出ます。

対処: ワークフローファイルに変更を加えて push すると GitHub が認識します。

```bash
for f in graphs response-time site summary update-template updates uptime; do
  printf '\n' >> .github/workflows/$f.yml
done
git commit -am "Register workflows" && git push
```

### 6-1. 監視の実行

1. `Actions → Uptime CI → Run workflow` を手動実行
2. 成功すると `history/*.yml` と `api/**` がコミットされる
3. `Actions → Summary CI` 実行後、README のステータス表と `graphs/` が更新される
4. Slack 通知のテスト:
   - `.upptimerc.yml` に存在しない URL を一時的に追加して push → 5分以内に Slack へ 🟥 通知 + Issue 起票
   - 確認後にその行を削除して push すると 🟩 復旧通知 + Issue 自動クローズ
   - Webhook 単体の疎通確認だけなら:
     ```bash
     curl -X POST -H 'Content-type: application/json' \
       --data '{"text":"MEDRT Upptime テスト通知"}' \
       'https://hooks.slack.com/services/XXX/YYY/ZZZ'
     ```

---

## 7. 運用メモ

| 項目 | 内容 |
| --- | --- |
| 監視間隔 | 設定値は5分（`uptime.yml` の cron `*/5 * * * *`）。ただし GitHub の schedule は保証されず、実績では間引かれます（`ema_sound_upptime` は約34回/日 = 平均40〜45分間隔）。検知の遅れを許容できない場合は外部の cron から `workflow_dispatch` を叩く運用を検討してください |
| ダウン判定 | HTTP ステータスが期待値以外、またはタイムアウト |
| 障害記録 | GitHub Issue として自動起票 → 復旧時に自動クローズ |
| 通知先 | Slack（Incoming Webhook）＋ Issue の Watch 通知 |
| 履歴データ | `history/<site>.yml`（コミット履歴として保持） |
| ワークフロー更新 | `update-template.yml` が毎週 Upptime 本体の更新を取り込む（`.github/workflows/*` は直接編集しない） |
| 設定変更 | `.upptimerc.yml` のみを編集 → push すると `Setup CI` がワークフローを再生成 |

### 監視対象の追加・変更

`.upptimerc.yml` の `sites` に追記して push するだけです。

```yaml
sites:
  - name: medrt_xxx_prd
    url: https://example.com/
    expectedStatusCodes:
      - 200
    maxResponseTime: 5000 # 任意: これを超えると「一部機能低下」扱い
```

### この監視でカバーできない範囲

外形監視のため、以下は検知できません。別途 VPS 側の対応が必要です。

- ディスク使用率・メモリ・CPU などのホストメトリクス
- MySQL コンテナ単体の異常（API が応答している限り検知不可）
- バッチ／cron の失敗

AWS 移行後は `medrt_cdk` の `monitoring-stack.ts`（CloudWatch + SNS）で上記をカバーする想定です。
