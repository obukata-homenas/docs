# 15. todo-app(やりたい事リスト専用サービス)

`server-dashboard` に埋め込まれていた「やりたい事」ページ(`backlog.md` の読み取り専用ビュー)を
独立させた、フルスクラッチのタスク管理サービス。

- リポジトリ: `obukata-homenas/todo-app`(`/opt/ops/apps/todo-app/`)
- 開発の経緯・要件・設計・決定事項: リポジトリ内 `planning/`(requirements.md /
  architecture-notes.md / backend-design.md / frontend-design.md / decisions.md)
- **デプロイ手順**: `planning/deployment.md`(このリポジトリ自体が正のデプロイ元。newdashと
  同じ方式で、server-configのミラー方式ではない)
- アクセス: `https://todo.obukata.uk`、Tailscale経由の非公開のみ(Obsidian等と同じ方式)
- データ: SQLite(`/srv/appdata/todo-app/data/`、既存の日次バックアップに自動的に乗る)

`docs/backlog.md` は当面並行運用し、移行が落ち着いたら正の座を退任する予定(backlog.md参照)。
