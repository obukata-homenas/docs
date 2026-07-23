# obukata-homenas/docs

obukata と AI エージェントが共有する運用ドキュメント。目次はこの3つだけ。

- **[`homenas/`](homenas/README.md)** — 家庭用ファイルサーバ(Mac mini + OpenMediaVault)自体の仕様・構築手順・運用ノウハウ。ハード構成やネットワーク、バックアップなど「サーバというインフラ」についての話はここ。
- **[`projects/`](projects/)** — 個々のプロジェクトごとのドキュメント(`projects/<project名>/`)。プロジェクト固有の話はここ。詳細な設計・決定事項は各プロジェクトのリポジトリ自体(`/opt/ops/apps/<project名>/`)に置かれていることが多く、ここはその入口。
- **[`ai-agent-policy.md`](ai-agent-policy.md)** — obukata と AI エージェントが会話・作業する上で、プロジェクトを問わず常に踏まえておくべき基本方針。

このリポジトリは `obukata` が管理し、`aiagent` はグループ `ops` の ACL で書き込み可能(実体との乖離が分かった場合に更新してよい。理由をコミットメッセージに残すこと)。ただし `/opt/ops/AGENTS.md` 自体は対象外。
