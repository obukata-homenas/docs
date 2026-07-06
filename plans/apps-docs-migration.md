# 移設プラン:全アプリ設定 + docs を /opt/ops・GitHub org へ

作成 2026-07-03。目的:homelab の各アプリ設定と docs を、共有エリア `/opt/ops` と
GitHub org `obukata-homenas` 配下で git 管理し、AI(aiagent)が運用できる状態にする。
※これはドラフト。obukata の承認・決定後に段階実行する。

## 現状(2026-07-06 更新)
- 移設済み:`server-dashboard`(→ `/opt/ops/apps/`、`obukata-homenas/server-dashboard`)、
  docs(→ `/opt/ops/docs/`、`obukata-homenas/docs`)。
- **6アプリとも `/opt/ops/apps/server-config/<svc>/` への設定取り込み・整合性確認まで完了**
  (`adguardhome`, `cloudflared`, `immich`, `nextcloud`, `npm`, `wireguard`)。2026-07-06 に
  `docker-compose.yml` 全文diffと `.env` キー名の差分を実機で確認し、**全アプリ差分ゼロ**を確認済み
  (詳細は Part B の Phase 2 参照)。
- これらは newdash と違い **third-party イメージ**。git 管理すべきは **設定(compose / env / 補助スクリプト)** であって、
  **データ(`/srv/appdata`・HDD の写真やファイル)ではない**。

## 方針(決めごと)
- **D1. git 管理は設定のみ**。データは対象外(巨大・git 不適)。【推奨・確定でよい】
- **D2. 秘密情報【決定:A、2026-07-03】**:**実際の秘密は root 専用のまま**(各サービスの `/srv/compose/<svc>/.env` に 600 root)、
  **`/opt/ops` には置かない**(=グループ ops の aiagent にも読ませない=影響範囲を維持)。git には `.env.example`(値伏せ雛形)だけ commit。
  - 秘密を持つのは cloudflared(TUNNEL_TOKEN)/immich(DB_PASSWORD)/nextcloud(MYSQL_*・NEXTCLOUD_ADMIN_PASSWORD)。
  - 平文の秘密は private リポでも GitHub に上げない・`/opt/ops` にも置かない、を鉄則。
- **D3. リポジトリ構成【決定:(b)、2026-07-03 承認】**:
  - インフラ設定系は **単一 `obukata-homenas/server-config` に app 別ディレクトリでまとめる**
    (proxy-net 等の相互依存があり一括で見たい/揃えたい)。
  - コードの `server-dashboard` は独立リポのまま(`obukata-homenas/server-dashboard`)。
- **D4. OMV との整合【Phase 0 で確定】**:`/srv/compose` は OMV Compose プラグインが root:root で管理・再適用する領域。
  移設パターン2案:
  - (i) **newdash 方式**:正=`/opt/ops`(git)、そこから `sudo <deploy スクリプト>` で `/srv/compose` へ反映。AI が設定編集→デプロイできる。
  - (ii) **バックアップのみ**:OMV UI を編集の主とし、`/opt/ops` へ git ミラー(保全目的)。AI は編集しない。
  - 各アプリが (i)/(ii) どちら向きかは Phase 0 の調査で判断。
- **D5. docs → org【推奨】**:`obukata/home-nas` を GitHub の **Transfer repository** で org `obukata-homenas` へ移管
  (履歴・URL リダイレクト保持)→ `/opt/ops/docs` の remote を張替え。新規作成+re-push より綺麗。

## Part A:docs を org へ(短い・先に片付ける)【完了 2026-07-03】
1. ✅ `obukata/home-nas` を org `obukata-homenas` へ Transfer + `docs` にリネーム(→ `obukata-homenas/docs`)。
2. ✅ `/opt/ops/docs` の remote を `git@github-private:obukata-homenas/docs.git` に張替え・fetch/push 疎通確認済み。
3. AGENTS.md の参照は相対パス(`docs/...`)なので変更不要。

## Part B:全アプリ設定を /opt/ops + org へ
- **Phase 0:棚卸し(1回・sudo 調査)【完了】** — 各サービスの compose/.env の場所、秘密の種類、
  デプロイ方法(OMV UI か wrapper か)、依存(proxy-net 等)、データパスを洗い出し、D3/D4 を確定。
  結果は `docs/plans/phase0-inventory.md`。
- **Phase 1:器の用意【完了】** — D3 の結論に沿って `obukata-homenas/server-config` を作成し
  `/opt/ops/apps/` 配下に配置、`.gitignore`(実 `.env`・秘密ファイル除外)を整備。6アプリ全ての
  `docker-compose.yml` + `.env.example` を初期構成として取り込み済み。
- **Phase 2:アプリ単位での整合性確認【完了・2026-07-06】**。推奨順(小・低リスクから):
  1. `cloudflared`(小・秘密=トークンのみ)
  2. `adguardhome`
  3. `npm`(Nginx Proxy Manager)
  4. `wireguard`
  5. `immich`(大)
  6. `nextcloud`(最重要・DB 秘密多い → 最後に慎重に)
  - 2026-07-06、obukata が実機で `sudo diff` を実行し、6アプリ全ての `docker-compose.yml`
    (nextcloudは `docker-compose.override.yml` も含む)と `.env` のキー名を
    `/opt/ops/apps/server-config/<svc>/` と比較 → **全アプリ差分ゼロ**を確認。
    既に稼働中のサービスの設定そのものなので「稼働確認」も実質的に満たされている。
    → 追加の commit は不要(ズレが無いため)。デプロイパターンは全アプリ **(ii) バックアップのみ**
    (OMV UI が編集の主、`/opt/ops` はミラー)。(i) newdash方式の自動デプロイ導入は見送り、
    必要になれば別途検討(sudo/Dockerアクセスの見直しと合わせて検討。backlog参照)。
- **Phase 3:仕上げ【未着手】** — AGENTS.md / docs に各アプリの運用手順を追記(既存の
  `docs/05`〜`docs/08`, `docs/12` 等がその役割を概ね果たしているため、追加が必要かは要確認)、
  旧配置の整理(該当なし、`/srv/compose` が元から唯一の実体)、全体稼働確認(Phase 2で実質完了)。

## 原則・リスク管理
- **秘密を平文で GitHub に上げない**(private でも)。`.env.example` 徹底、commit 前に人間が確認。
- **データには触れない**(写真・ファイル・DB 本体)。
- **1アプリずつ・稼働確認しながら**。`nextcloud`/`immich` は特に慎重(データは別管理)。
- push(egress)は aiagent の autoMode.environment 済で通るが、**秘密混入の最終確認は人間**が行う。

## 進め方
各フェーズ/各アプリの後で obukata に確認 → 次へ(1ステップずつ)。まず Part A、次に Part B の Phase 0。
