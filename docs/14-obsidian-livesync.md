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

1. Obsidian → 設定 → コミュニティプラグイン → 「Self-hosted LiveSync」を検索してインストール・有効化
2. プラグイン設定でサーバURL `https://obsidian.obukata.uk`、ユーザー名/パスワードに
   `COUCHDB_USER`/`COUCHDB_PASSWORD` を入力
3. **エンドツーエンド暗号化パスフレーズを設定する(強く推奨)**。CouchDB はノートの中身を
   暗号化せず平文で保存し、日次バックアップにもそのまま平文で入るため、これを設定しないと
   ディスクやバックアップに直接アクセスできる人には内容が読めてしまう。パスフレーズを設定
   すれば、サーバに保存される時点で暗号化された状態になる。**パスフレーズを忘れると復元
   できなくなる**ので、忘れない場所に控えておくこと。
4. 他の端末でも同じ URL・認証情報・パスフレーズで設定(1台目が同期の起点になる)
5. Vault が大きい場合、初回同期は時間がかかる(想定内)

## 7. 任意の追加ハードニング

上記で必要十分だが、さらに絞りたい場合:

- 管理者アカウント(`COUCHDB_USER`/`COUCHDB_PASSWORD`)を LiveSync プラグインにそのまま使う
  代わりに、Obsidian 用データベースにのみ権限を持つ専用ユーザーを CouchDB 側で作成して使う
  (漏洩時の影響範囲を「管理者権限全体」から「このデータベースだけ」に縮小できる)。
