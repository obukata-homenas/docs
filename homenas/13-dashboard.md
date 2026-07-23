# 13. ダッシュボード (Homepage)

前: [12. Immich の導入](12-immich.md)

## 目的

各サービスへの入口＋ステータス＋システム情報を1画面に集約するランチャー型ダッシュボード
[Homepage](https://gethomepage.dev/) を Docker で導入する。`https://dashboard.home.lan`（LAN内のみ。外出先は WireGuard 経由）でアクセスする。

## 構成

| 項目 | 値 |
|---|---|
| イメージ | `ghcr.io/gethomepage/homepage:latest` |
| compose | `/srv/compose/dashboard/` |
| 設定ファイル | `/srv/appdata/homepage/config/`（`settings.yaml` / `services.yaml` / `widgets.yaml`） |
| コンテナ内ポート | 3000（直アクセス確認用に host 3001 を一時公開） |
| 公開 | NPM で `dashboard.home.lan` → `homepage:3000`（LAN 内のみ） |

## 手順

### 13-1. ディレクトリ作成

```bash
sudo mkdir -p /srv/compose/dashboard /srv/appdata/homepage/config
```

### 13-2. compose と設定ファイルの配置

`/srv/compose/dashboard/docker-compose.yml`:

```yaml
services:
  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    container_name: homepage
    restart: unless-stopped
    environment:
      # Homepage は許可ホストの明示が必須 (セキュリティ)
      HOMEPAGE_ALLOWED_HOSTS: dashboard.home.lan,192.168.3.19:3001
    ports:
      - "3001:3000"   # 動作確認用の直アクセス。NPM 経由が安定したら外してよい
    volumes:
      - /srv/appdata/homepage/config:/app/config
      # ストレージ使用量表示用 (read-only)
      - /srv:/mnt/ssd:ro
      - /srv/dev-disk-by-uuid-3a9a7624-afbb-4993-a1dd-38e2bd1546f5:/mnt/hdd:ro
    networks:
      - proxy-net

networks:
  proxy-net:
    external: true
```

`/srv/appdata/homepage/config/settings.yaml`:

```yaml
title: ホームサーバ ダッシュボード
theme: dark
headerStyle: clean
layout:
  写真・ファイル:
    style: row
    columns: 2
  ネットワーク・管理:
    style: row
    columns: 3
```

`/srv/appdata/homepage/config/services.yaml`:

```yaml
- 写真・ファイル:
    - Nextcloud:
        href: https://nextcloud.home.lan
        description: ファイル同期・共有
        siteMonitor: http://nextcloud:80
    - Immich:
        href: https://immich.home.lan
        description: 写真・動画
        siteMonitor: http://immich_server:2283

- ネットワーク・管理:
    - AdGuard Home:
        href: http://192.168.3.19:8082
        description: LAN内DNS・広告ブロック
        siteMonitor: http://192.168.3.19:8082
    - Nginx Proxy Manager:
        href: http://192.168.3.19:81
        description: リバースプロキシ
        siteMonitor: http://192.168.3.19:81
    - OpenMediaVault:
        href: http://192.168.3.19:8181
        description: NAS 管理画面
        siteMonitor: http://192.168.3.19:8181
```

`/srv/appdata/homepage/config/widgets.yaml`:

```yaml
- resources:
    label: システム
    cpu: true
    memory: true
- resources:
    label: SSD
    disk: /mnt/ssd
- resources:
    label: HDD
    disk: /mnt/hdd
- datetime:
    text_size: xl
    format:
      dateStyle: long
      timeStyle: short
      hourCycle: h23
```

> `siteMonitor` は up/down ランプ用。`proxy-net` 上のコンテナはコンテナ名（`nextcloud:80` 等）、それ以外はホスト IP:port で監視する。自己署名 https を直接監視すると TLS 検証で落ちるため、監視先は http / 内部ポートを使う。

### 13-3. 起動

```bash
sudo -i
cd /srv/compose/dashboard
docker compose up -d
docker compose logs -f homepage
# "ready" 表示後、Ctrl+C → exit
exit
```

確認用に `http://192.168.3.19:3001/` を開く（許可ホストに含めてある）。サービス一覧とシステム/ストレージ情報が出れば OK。

### 13-4. AGH に DNS エントリを追加

AdGuard Home (`http://192.168.3.19:8082`) → **Filters → DNS rewrites → Add DNS rewrite**
`dashboard.home.lan` → `192.168.3.19`

### 13-5. NPM にプロキシホストを追加

Nginx Proxy Manager (`http://192.168.3.19:81`) → **Proxy Hosts → Add Proxy Host**

| 項目 | 値 |
|---|---|
| Domain Names | `dashboard.home.lan` |
| Scheme | `http` |
| Forward Hostname / IP | `homepage` |
| Forward Port | `3000` |
| Websockets Support | ON |

SSL タブ → 既存の自己署名証明書を選択（または `dashboard.home.lan` 用に新規作成）→ Force SSL ON。

確認: `https://dashboard.home.lan/` が表示できれば完了。安定したら compose の `ports: 3001:3000` は外してよい（その場合 `HOMEPAGE_ALLOWED_HOSTS` から `192.168.3.19:3001` も外す）。

## 後から拡張できること（任意）

- **サービス専用ウィジェット**: Nextcloud / AdGuard 等は API キーを設定すると、ストレージ使用量・ブロック数などの統計を Homepage 上に表示できる（各サービスでキー発行 → `services.yaml` の該当サービスに `widget:` を追加）。キーは設定ファイルに平文で入るため取り扱い注意。
- **Docker 連携**: `docker.sock` を read-only マウントすると各コンテナの稼働状態を自動表示できる。ソケット露出のリスクがあるため、入れる場合は `:ro` で。

## トラブルシュート

- **`Host validation failed` / 画面が出ない**: `HOMEPAGE_ALLOWED_HOSTS` にアクセス中のホスト（`dashboard.home.lan` や `192.168.3.19:3001`）が入っているか確認 → 直して `docker compose up -d`
- **ストレージが出ない**: `/mnt/ssd` `/mnt/hdd` のマウント（compose の volumes）と `widgets.yaml` の `disk:` パス一致を確認
- **up/down が赤**: `siteMonitor` の宛先がコンテナから到達できるか（proxy-net 上はコンテナ名、それ以外はホスト IP:port）

## 次へ

構築完了後の運用は [operations.md](operations.md) を参照。
