# 14. Obsidian のセルフホスト(Self-hosted LiveSync)

Obsidian のノートを Mac / iPhone / Android 間で同期する。有料の Obsidian Sync は使わず、
homenas 上に自前の CouchDB を立て、Obsidian の community plugin **「Self-hosted LiveSync」**
経由で同期する。

設定ファイル(compose / `.env.example`)は `obukata-homenas/server-config` リポジトリの
`obsidian/` に git 管理されている(`/opt/ops/apps/server-config/obsidian/`)。本ドキュメントは
それを実際にサーバへデプロイする手順。

## 前提

- 秘密情報(CouchDB 管理者パスワード)は `/srv/compose/obsidian/.env`(root 専用・600)にのみ置く。
  `/opt/ops` 側には `.env.example`(値を伏せた雛形)しかない。
- 公開はしない。アクセスは **Tailscale 経由の非公開のみ**(nextcloud/immich/dashboard と同じ方式)。
- `*.obukata.uk` の Let's Encrypt ワイルドカード証明書と AdGuard Home の `*.obukata.uk` ワイルドカード
  DNS rewrite は既に存在するため、**新規の証明書取得・DNS 追加は不要**。
- CouchDB のポートはホストの `127.0.0.1` にのみ公開している(`0.0.0.0` に公開すると、Docker が
  ufw より先に評価される iptables ルールを追加するため、Tailscale限定のつもりが LAN 全体から
  平文で到達可能になってしまう。レビューで確認済み)。サーバ内からの `curl`(ブートストラップ・
  動作確認用)はこれで問題なく動くが、他端末からは NPM 経由の `https://obsidian.obukata.uk` の
  みが到達経路になる。

## 1. コンテナのデプロイ

```bash
sudo mkdir -p /srv/compose/obsidian
sudo cp /opt/ops/apps/server-config/obsidian/docker-compose.yml /srv/compose/obsidian/
sudo cp /opt/ops/apps/server-config/obsidian/.env.example /srv/compose/obsidian/.env

# 実際の値を入れる(COUCHDB_USER / COUCHDB_PASSWORD)
sudo nano /srv/compose/obsidian/.env
sudo chmod 600 /srv/compose/obsidian/.env
sudo chown root:root /srv/compose/obsidian/.env

# データ用ディレクトリを作成し、CouchDB コンテナの実行ユーザー(uid/gid 5984)に渡す
sudo mkdir -p /srv/appdata/obsidian/data /srv/appdata/obsidian/etc-local.d
sudo chown -R 5984:5984 /srv/appdata/obsidian
# uid/gid が違う場合は `docker run --rm couchdb:3.4.3 id couchdb` で確認

cd /srv/compose/obsidian
sudo docker compose up -d
sudo docker compose logs -f couchdb
# "Apache CouchDB has started" が出て、権限エラーが無ければ Ctrl+C
```

## 2. CouchDB の初回セットアップ(1回だけ)

Self-hosted LiveSync が要求する設定(シングルノード化・システム DB 作成・CORS・認証必須化)を行う。
どちらの方法でも良い(再実行しても安全):

**方法A(LiveSync 本家のスクリプトを使う)**

```bash
# 先に内容を確認してから実行する(admin 権限で叩くスクリプトなので中身を見ておく)
curl -s https://raw.githubusercontent.com/vrtmrz/obsidian-livesync/main/utils/couchdb/couchdb-init.sh

# 確認後、実行
curl -s https://raw.githubusercontent.com/vrtmrz/obsidian-livesync/main/utils/couchdb/couchdb-init.sh \
  | hostname=http://127.0.0.1:5984 username=<COUCHDB_USER> password=<COUCHDB_PASSWORD> bash
```

**方法B(手動で curl を叩く)**

```bash
U=<COUCHDB_USER>; P=<COUCHDB_PASSWORD>; H=http://127.0.0.1:5984

# シングルノード化 + システムDB(_users/_replicator/_global_changes)作成
curl -X POST -H 'Content-Type: application/json' "$H/_cluster_setup" -u "$U:$P" -d \
  "{\"action\":\"enable_single_node\",\"username\":\"$U\",\"password\":\"$P\",\"bind_address\":\"0.0.0.0\",\"port\":5984,\"singlenode\":true}"

curl -X PUT "$H/_node/_local/_config/chttpd_auth/require_valid_user" -u "$U:$P" -d '"true"'
curl -X PUT "$H/_node/_local/_config/chttpd/require_valid_user"      -u "$U:$P" -d '"true"'
curl -X PUT "$H/_node/_local/_config/chttpd/enable_cors"             -u "$U:$P" -d '"true"'
curl -X PUT "$H/_node/_local/_config/httpd/enable_cors"              -u "$U:$P" -d '"true"'
curl -X PUT "$H/_node/_local/_config/cors/credentials"               -u "$U:$P" -d '"true"'
curl -X PUT "$H/_node/_local/_config/cors/origins"                   -u "$U:$P" -d '"app://obsidian.md,capacitor://localhost,http://localhost"'
curl -X PUT "$H/_node/_local/_config/chttpd/max_http_request_size"   -u "$U:$P" -d '"4294967296"'
curl -X PUT "$H/_node/_local/_config/couchdb/max_document_size"      -u "$U:$P" -d '"50000000"'
curl -X PUT "$H/_node/_local/_config/httpd/WWW-Authenticate"         -u "$U:$P" -d '"Basic realm=\"couchdb\""'
```

(最後の `WWW-Authenticate` は本家スクリプトにも含まれる設定。方法Aを使えばこの1行も含めて自動で入る。)

`/opt/couchdb/etc/local.d` を volume でマウントしているので、ここで設定した内容は
コンテナの再作成・イメージ更新後も保持される(volume を消さない限り再実行不要)。

## 3. 動作確認

```bash
curl -s -u <COUCHDB_USER>:<COUCHDB_PASSWORD> http://127.0.0.1:5984/_up
# => {"status":"ok",...}

curl -s -u <COUCHDB_USER>:<COUCHDB_PASSWORD> http://127.0.0.1:5984/_all_dbs
# => ["_replicator","_users",...] が含まれていること
```

Fauxton(管理UI): サーバ内からは `http://127.0.0.1:5984/_utils`、NPM設定後は他端末からも
`https://obsidian.obukata.uk/_utils` でアクセス可能(ポートをループバック限定にしているため、
`<Tailscale IP>:5984` への直接アクセスはできない)。

## 4. NPM(Nginx Proxy Manager)の設定

NPM 管理画面(`http://<Tailscale IP>:81/`)→ **Hosts → Proxy Hosts → Add Proxy Host**

| 項目 | 値 |
|---|---|
| Domain Names | `obsidian.obukata.uk` |
| Scheme | `http` |
| Forward Hostname / IP | `obsidian`(コンテナ名。`proxy-net` 経由で到達) |
| Forward Port | `5984` |
| Websockets Support | **On**(LiveSync の同期は長時間接続を使うため) |
| Block Common Exploits | On |

**SSL タブ**: SSL Certificate は既存の `*.obukata.uk` ワイルドカード証明書を選択(新規発行しない)。
Force SSL: On、HTTP/2: On。

Advanced タブに以下を追記(添付ファイルが大きくなる場合に備えたnginx側の上限):
```
client_max_body_size 50M;
```

AdGuard Home 側は `*.obukata.uk` のワイルドカード rewrite が既に存在するので追加設定不要。
念のため管理画面で `obsidian.obukata.uk` が解決できることだけ確認する。

## 5. バックアップ

`homenas-backup.sh` は `${SSD_ROOT}/appdata/` を丸ごと rsync しているため、
`/srv/appdata/obsidian/` は追加設定なしで自動的に日次バックアップの対象になる。

## 6. 各端末の Obsidian 設定

実際に3台(iPhone/Mac)構築してハマったポイントを反映済み。次の端末を追加するときはこの章だけ
見れば迷わないはず。

### 6-0. 前提: Tailscale が必須

`obsidian.obukata.uk` は公開DNSには存在せず、AdGuard Home が「Tailscale経由で来た問い合わせ」
にだけサーバーのTailscale IPを返す仕組み。**同じLAN内にいてもTailscaleに接続していない端末
からは繋がらない**(DNSが引けない上に、Tailscale IP自体そもそも経路的に届かない)。

新しい端末を追加する前に:
1. その端末にTailscaleアプリをインストールし、他の端末と同じアカウントでログイン・接続
2. DNSの「Override local DNS」設定はTailscaleのアカウント側(管理コンソール)で1回設定済みなので、
   新しい端末は同じアカウントでログインするだけで自動的に反映される(端末ごとの追加設定は不要)
3. 確認: ブラウザで `https://obsidian.obukata.uk/_utils` を開いてログイン画面が出ればOK

### 6-1. 1台目(最初の端末)のセットアップ

初回はサーバー情報を手入力する。

1. Obsidian → 設定 → コミュニティプラグイン → 「Self-hosted LiveSync」を検索してインストール・有効化
2. プラグインの Setup ウィザードで「Start」(新規セットアップ)を選ぶ。既存の Setup URI は無いので
   「Connect with Setup URI」ではなくこちら
3. Remote Type は `CouchDB` を選択
4. 接続情報を入力:
   - Server URI: `https://obsidian.obukata.uk`
   - Username / Password: `.env` の `COUCHDB_USER` / `COUCHDB_PASSWORD`
   - Database name: 好きな名前(小文字・記号なし。例: `obsidian-vault`)。事前にCouchDB側で
     作成しておく必要はなく、自動的に作られる
   - Use Internal API: **OFFのままでよい**(CORSがどうしても解決できない時だけの緊急手段)
5. **エンドツーエンド暗号化パスフレーズを設定する(強く推奨)**。CouchDB はノートの中身を
   暗号化せず平文で保存し、日次バックアップにもそのまま平文で入るため、これを設定しないと
   ディスクやバックアップに直接アクセスできる人には内容が読めてしまう。**パスフレーズを
   忘れると復元できなくなる**ので、パスワードマネージャー等に必ず控えておくこと
6. **Obfuscate Properties にチェックを入れる(推奨)**。ファイルパス・ファイル名・作成/更新日時
   などのメタデータをサーバー上で隠せる(ノートの中身はE2E暗号化で守られるが、メタデータは
   別。副作用はほぼ無い)
7. 初回のデータ同期方法を聞かれる:
   - 全端末ともVaultが空(これから新規に使う)なら **「Compare time and take newer」を選べば
     どれを選んでも安全**(消えるデータが無いため)
   - もし既存のノートが入っている端末が1台でもある場合は、**そのノートが入っている端末を
     必ず1台目にする**。空のVaultを「Overwrite all with remote files」的な選択肢で
     同期してしまうと、既存ノートが消える恐れがある
8. Conflict/削除設定: 「Delete local files if deleted on remote」を推奨(普通の同期らしい
   挙動。Obsidian自体がゴミ箱機能を持っているので誤削除の保険にもなる)
9. 「Fetch Remote Configuration Failed」という警告が出ることがあるが、**1台目では正常**
   (サーバー側にまだ取得すべき設定が無いだけ)。「Skip and proceed」で進めてよい

### 6-2. 2台目以降のセットアップ(Setup URI を使う)

1台目の設定が終わったら、2台目以降は Setup URI を使うと手入力が要らず楽。

1. **1台目**のプラグイン設定 → Setupウィザード(🧙アイコン)→「現在の設定をSetup URIとして
   コピー」的な項目を選ぶ
2. URI暗号化用のパスフレーズを聞かれる(**手順6-1の⑤の暗号化パスフレーズとは別物**。
   この受け渡し専用の一時的なものでよい)
3. Setup URIがクリップボードにコピーされる。新しい端末に転送する(iPhone/Macならユニバーサル
   クリップボード、それ以外は自分宛メッセージ等で)
4. **新しい端末**でプラグインをインストール・有効化 → Setupウィザードで「Connect with Setup
   URI」を選択 → 貼り付け → 手順②のパスフレーズを入力
5. サーバーURL・認証情報・データベース名・E2E暗号化パスフレーズが自動入力される
6. 初回同期方法を聞かれたら、**サーバーから取得する(Fetch)方向の選択肢を選ぶ**
   (この新しい端末のVaultは空で、サーバー側に既にデータがあるはず。逆(空を優先)を選ぶと
   サーバー側のデータを消しかねないので注意)

### 6-3. 設定は終わったのに同期されない時(重要)

セットアップ直後、ステータスバーが **「Sync: ↑0 ↓0」のまま変化しない**(エラーは出ない)
ことがある。これは既知の挙動で、初回は手動で同期を1回蹴ってやる必要がある:

1. コマンドパレットを開く(Mac: `Cmd+P`、スマホはObsidianの検索/コマンドアイコン)
2. 「**Self-hosted LiveSync: Replicate now**」を実行
3. それでも動かなければ「**Self-hosted LiveSync: Toggle LiveSync**」を1回OFF→ONし直す
   (実体験: 初期状態でLiveSyncがEnableになっていないことがあった)

### 6-4. トラブルシューティング早見表

| 症状 | 原因・対処 |
|---|---|
| `DANGER Failed to connect to the server` | まずTailscaleが接続されているか確認(§6-0)。次にブラウザで `https://obsidian.obukata.uk/_utils` が開けるか確認 |
| `Fetch Remote Configuration Failed` | 1台目では正常。「Skip and proceed」でよい |
| ステータスが `Sync: ↑0 ↓0` のまま | §6-3 参照(Replicate now / Toggle LiveSync) |
| Fauxtonでデータが増えているか確認したい | `https://obsidian.obukata.uk/_utils` → Databases → 対象データベース名 → ドキュメント件数を確認。Obfuscate Properties ON だとファイル名等は読めないが正常 |

## 7. 任意の追加ハードニング

上記で必要十分だが、さらに絞りたい場合:

- 管理者アカウント(`COUCHDB_USER`/`COUCHDB_PASSWORD`)を LiveSync プラグインにそのまま使う
  代わりに、Obsidian 用データベースにのみ権限を持つ専用ユーザーを CouchDB 側で作成して使う
  (漏洩時の影響範囲を「管理者権限全体」から「このデータベースだけ」に縮小できる)。
