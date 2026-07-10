# やりたいこと(バックログ) — 過去ログ

<!--
2026-07-10、todo-app(https://todo.obukata.uk)へ全項目を移行済み。
以後の「やりたい事」の管理はtodo-app側が正。このファイルは移行前の履歴として残す。
-->

## 完了

### docs/backlog.md の内容を todo-app へ移行する
- 優先度: 中
- 背景: 「やりたい事リスト」独立サービス(todo-app)がデプロイ完了し稼働中。decisions.mdの方針通り、`docs/backlog.md` は当面並行運用し、実際の項目移行はまだこれからだった。
- 次の一歩: 完了(2026-07-10)。今やる/近いうち/いつかの計10件をtodo-appへ移行(`POST /api/items` + AI_PROGRESS_TOKEN経由で移行元メモを1件ずつ添付)。以後、本ファイルは過去ログ・参照専用とし、新規の「やりたい事」はtodo-app側で管理する。

### バイブコーディングをマルチエージェント開発に進化させる
- 優先度: 中
- 背景: 今は技術スタックや実装方法まで指定するスタイル。ゴールだけ伝えて、プランナー/エンジニア/デザイナー的なエージェントに分担させて自律開発する手法を試したい。
- 次の一歩: 完了(2026-07-10)。todo-appプロジェクトで実践。マネージャー(要件ヒアリング・調整)/フロントエンド(見た目はobukataと直接やり取り)/バックエンド(データモデル・API)/QA(独立レビュー)の4エージェント体制で、要件ヒアリングから実装・デプロイまで一通り完走。aiagentは各エージェントの結果を統合・報告する「代表」役に徹する運用。

### 「やりたい事リスト」を独立したサービスとして切り出す
- 優先度: 中
- 背景: 現状ダッシュボードの「やりたい事」ページは `docs/backlog.md` をそのまま表示しているだけで見づらい。Todoアプリ的、あるいはチケット発行するタスク管理サービス的な、専用の見やすいサービスとして独立させたい。Claudeにフルスクラッチで作ってもらうか、既存のセルフホスト可能なタスク管理サービス(例: Vikunja, Planka 等)を導入するか、方針から検討したい。
- 次の一歩: 完了(2026-07-10)。マネージャー・frontend・backend・QAの4エージェント体制でフルスクラッチ開発(既存ツールは見た目の好みが合わず不採用)。要件ヒアリング→デザイン3案→実装→QAレビュー→デプロイまで完了し、`https://todo.obukata.uk` で稼働中(Tailscale経由の非公開アクセス)。リポジトリは `obukata-homenas/todo-app`。

### server-config のテストブランチ/PR の片付け
- 優先度: 低
- 背景: 2026-07-05 に branch→push→PR フローの動作確認で作った `aiagent/test-pr` ブランチが残っていた。作業ツリーも main ではなくそのブランチのままになっていた。
- 次の一歩: 完了(2026-07-07)。マージ済みの `aiagent/add-obsidian-livesync`・`aiagent/fix-obsidian-couchdb-exposure` とあわせてリモートブランチ3つを削除。`main` に復帰。ローカルの `aiagent/test-pr`(未マージのため強制削除がaiagent側では権限拒否対象)はobukataが手動で削除。

### アプリ設定移設プラン(Part B)の続行判断
- 優先度: 中
- 背景: 2026-07-03 に承認された移設プラン(`docs/plans/apps-docs-migration.md`)のうち、Part A(docs の org 移設)は完了、Part B の Phase 0(棚卸し)も完了済みだった。Phase 1(`server-config` リポジトリの器づくり)・Phase 2(6アプリの整合性確認)が残っていた。
- 次の一歩: 完了(2026-07-06)。Phase 1は実は初期構成の時点で6アプリ全ての `docker-compose.yml`+`.env.example` が既に取り込まれていたと判明。Phase 2はobukataが実機で `sudo diff` を実行し、6アプリ(adguardhome/cloudflared/immich/nextcloud/npm/wireguard)全てで `/opt/ops/apps/server-config` と実体(`/srv/compose`)の差分ゼロを確認。追加のcommitは不要だった。デプロイパターンは全アプリ(ii)バックアップのみ(OMV UIが編集の主)で確定。詳細は `docs/plans/apps-docs-migration.md`。残るPhase 3(仕上げ)は優先度低のため様子見。

### Tailscale を導入して VPN を快適化(非公開アクセスの土台)
- 優先度: 中
- 背景: 外出先から Immich/Nextcloud アプリ・dashboard に繋ぎたい＆WireGuardのフルトンネルで帰宅時にネットが切れる問題を解消したかった。
- 次の一歩: 完了(2026-06-27)。サーバ＋スマホ/PC に Tailscale 導入(squib02@gmail.com)。スプリットトンネルで常時ONでも通常ネットOK・外出先から接続OKを実証。Tailscale DNS=AdGuard(100.72.254.73)に「Override local DNS」→全端末で広告ブロック＆内部名解決(バイパスは Tailscale OFF)。AdGuardに `*.obukata.uk`→100.72.254.73 のrewrite、NPMで `*.obukata.uk` のLet's Encryptワイルドカード証明書(CloudflareトークンでDNS-01)＋Proxy Host(immich/nextcloud/dashboard.obukata.uk)。Nextcloudは override.yml の OVERWRITEHOST 削除で多ドメイン対応。アプリは `https://〜.obukata.uk` で設定・動作確認済み。subnet router / WireGuard撤去 は任意で後日。

### Cloudflare Tunnel で自宅サーバを公開(セキュリティ最優先)
- 優先度: 中
- 背景: ポート開放せず外部公開＋ガッチガチ認証をやりたかった。
- 次の一歩: 完了(2026-06-26)。独自ドメイン **obukata.uk** を Cloudflare で取得、Zero Trust チーム `obukata`、トンネル `homelab` を作成、cloudflared を Docker(proxy-net)で起動、**dash.obukata.uk → newdash** を公開し **Cloudflare Access(Allow: squib02@gmail.com のみ)** で施錠まで検証OK。ただし方針転換:Immich/Nextcloud のアプリは Access で壊れる＆管理系は非公開が安全なので、当面は **VPN(Tailscale)前提の非公開運用**に。Cloudflare/ドメインは将来の公開サービス用に温存(cloudflared コネクタは稼働したまま=いつでも再利用可)。

### スマホで仕事できる体制を作る
- 優先度: 中
- 背景: スマホ(Termius)→ホームサーバ→tmux でこのセッション、という流れ。権限コマンドの実行・コピペが大変で、内容確認もできずブルシット感が強かった。
- 次の一歩: 完了(2026-06-25)。sudoスコープ化で権限コマンドは Claude が直接実行(手打ち/確認の手間ゼロ)、tmuxコピーは OSC52 整備、dashboard.home.lan も iPad から閲覧可(Wi-Fi の DNS を 192.168.3.19 に手動設定)。日本語IMEは Ghostty/WezTerm 等を検討したが Termius が最もマシと確定し許容。

### tmux のコピーモードを直す(キーボードで選択コピー)
- 優先度: 高
- 背景: tmux 上の文字をうまくコピーできない。Shift＋ドラッグ頼みで、キーボードでの選択コピーが効かなかった。スマホ作業の土台にもなる。
- 次の一歩: 完了(2026-06-25)。~/.tmux.conf に mode-keys vi / mouse on / set-clipboard on / terminal-features clipboard / v・y・C-v バインドを設定。SSH越しのMacクリップボードは Warp が OSC52 非対応でNGだったため Ghostty に変更して解決(サーバに xterm-ghostty terminfo も導入)。

### サーバのSMART監視をSlack通知化
- 優先度: 中
- 背景: ドライブ故障を事前に察知したい(外付けが壊れていた件の再発防止)
- 次の一歩: 完了(smartd + smart-slack.sh、2026-06-24 稼働)
