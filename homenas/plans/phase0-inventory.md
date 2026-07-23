# Phase 0 棚卸し(自動生成 / 秘密の値は伏せてある)

対象: adguardhome cloudflared immich nextcloud npm wireguard

## adguardhome
```
# files:
total 16
drwxr-sr-x 2 root users 4096 Apr 22 21:21 .
drwx--S--- 9 root users 4096 Jun 26 22:14 ..
-rw-r--r-- 1 root users  326 Apr 22 21:21 docker-compose.yml
-rw-r--r-- 1 root users   28 Apr 22 21:20 .env

# docker-compose.yml (secret値は伏せ):
services:
  adguardhome:
    image: adguard/adguardhome:latest
    container_name: adguardhome
    restart: unless-stopped
    network_mode: host
    environment:
      TZ: ${TZ}
    volumes:
      - ${SSD_ROOT}/appdata/adguardhome/work:/opt/adguardhome/work
      - ${SSD_ROOT}/appdata/adguardhome/conf:/opt/adguardhome/conf

# .env の KEY 一覧 (値は伏せ):
SSD_ROOT=<hidden>
TZ=<hidden>
```

## cloudflared
```
# files:
total 16
drwxr-xr-x 2 root root  4096 Jun 26 22:14 .
drwx--S--- 9 root users 4096 Jun 26 22:14 ..
-rw-r--r-- 1 root root   355 Jun 26 22:14 docker-compose.yml
-rw------- 1 root root   198 Jun 26 22:14 .env

# docker-compose.yml (secret値は伏せ):
services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: unless-stopped
    # トンネルトークンは同ディレクトリの .env (TUNNEL_TOKEN) から注入
    command: tunnel --no-autoupdate run --token ${TUNNEL_TOKEN}
    networks:
      - proxy-net
networks:
  proxy-net:
    external: true

# .env の KEY 一覧 (値は伏せ):
TUNNEL_TOKEN=<hidden>
```

## immich
```
# files:
total 16
drwxr-sr-x 2 root users 4096 Jun 13 16:31 .
drwx--S--- 9 root users 4096 Jun 26 22:14 ..
-rw-r--r-- 1 root users 1673 Jun 13 16:51 docker-compose.yml
-rw------- 1 root users  255 Jun 13 16:30 .env

# docker-compose.yml (secret値は伏せ):
name: immich

services:
  immich-server:
    container_name: immich_server
    image: ghcr.io/immich-app/immich-server:${IMMICH_VERSION:-release}
    restart: always
    env_file:
      - .env
    volumes:
      - ${UPLOAD_LOCATION}:/usr/src/app/upload
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "2283:2283"
    depends_on:
      - redis
      - database
    networks:
      - internal
      - proxy-net
    healthcheck:
      disable: false

  immich-machine-learning:
    container_name: immich_machine_learning
    image: ghcr.io/immich-app/immich-machine-learning:${IMMICH_VERSION:-release}
    restart: always
    env_file:
      - .env
    volumes:
      - model-cache:/cache
    networks:
      - internal
    healthcheck:
      disable: false

  redis:
    container_name: immich_redis
    image: redis:6.2-alpine
    restart: always
    networks:
      - internal
    healthcheck:
      test: redis-cli ping || exit 1

  database:
    container_name: immich_postgres
    image: tensorchord/pgvecto-rs:pg14-v0.2.0
    restart: always
    environment:
      POSTGRES_PASSWORD: <REDACTED>
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_DB: ${DB_DATABASE_NAME}
      POSTGRES_INITDB_ARGS: '--data-checksums'
    volumes:
      - ${DB_DATA_LOCATION}:/var/lib/postgresql/data
    networks:
      - internal
    command: >-
      postgres
      -c shared_preload_libraries=vectors.so
      -c 'search_path="$$user", public, vectors'
      -c logging_collector=on
      -c max_wal_size=2GB
      -c shared_buffers=512MB
      -c wal_compression=on

volumes:
  model-cache:

networks:
  internal:
    driver: bridge
  proxy-net:
    external: true

# .env の KEY 一覧 (値は伏せ):
IMMICH_VERSION=<hidden>
UPLOAD_LOCATION=<hidden>
DB_DATA_LOCATION=<hidden>
DB_PASSWORD=<hidden>
DB_USERNAME=<hidden>
DB_DATABASE_NAME=<hidden>
TZ=<hidden>
```

## nextcloud
```
# files:
total 28
drwxr-xr-x 2 root root  4096 Jun 27 18:51 .
drwx--S--- 9 root users 4096 Jun 26 22:14 ..
-rw-r--r-- 1 root root   115 Jun 27 18:51 docker-compose.override.yml
-rw-r--r-- 1 root root   155 Apr 22 22:14 docker-compose.override.yml.bak
-rw-r--r-- 1 root root  1440 Apr 19 21:56 docker-compose.yml
-rw------- 1 root root   377 Apr 19 21:55 .env
-rw------- 1 root root   377 Apr 19 21:49 .env.bak

# docker-compose.yml (secret値は伏せ):
services:
  db:
    image: mariadb:11
    container_name: nextcloud-db
    restart: unless-stopped
    command:
      - --transaction-isolation=READ-COMMITTED
      - --log-bin=binlog
      - --binlog-format=ROW
    environment:
      MYSQL_ROOT_PASSWORD: <REDACTED>
      MYSQL_PASSWORD: <REDACTED>
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
    volumes:
      - ${SSD_ROOT}/appdata/mariadb:/var/lib/mysql
    networks:
      - internal

  redis:
    image: redis:7-alpine
    container_name: nextcloud-redis
    restart: unless-stopped
    volumes:
      - ${SSD_ROOT}/appdata/redis:/data
    networks:
      - internal

  app:
    image: nextcloud:29-apache
    container_name: nextcloud
    restart: unless-stopped
    depends_on:
      - db
      - redis
    environment:
      MYSQL_HOST: db
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: <REDACTED>
      REDIS_HOST: redis
      NEXTCLOUD_ADMIN_USER: ${NEXTCLOUD_ADMIN_USER}
      NEXTCLOUD_ADMIN_PASSWORD: <REDACTED>
      NEXTCLOUD_TRUSTED_DOMAINS: ${NEXTCLOUD_TRUSTED_DOMAINS}
    volumes:
      - ${SSD_ROOT}/appdata/nextcloud/html:/var/www/html
      - ${HDD_ROOT}/appdata/nextcloud/data:/var/www/html/data
    ports:
      - "8080:80"
    networks:
      - internal
      - proxy-net

networks:
  internal:
    driver: bridge
  proxy-net:
    external: true

# .env の KEY 一覧 (値は伏せ):
SSD_ROOT=<hidden>
HDD_ROOT=<hidden>

MYSQL_ROOT_PASSWORD=<hidden>
MYSQL_PASSWORD=<hidden>
MYSQL_DATABASE=<hidden>
MYSQL_USER=<hidden>

NEXTCLOUD_ADMIN_USER=<hidden>
NEXTCLOUD_ADMIN_PASSWORD=<hidden>

NEXTCLOUD_TRUSTED_DOMAINS=<hidden>


```

## npm
```
# files:
total 16
drwxr-xr-x 2 root root  4096 Apr 22 21:55 .
drwx--S--- 9 root users 4096 Jun 26 22:14 ..
-rw-r--r-- 1 root root   365 Apr 22 21:52 docker-compose.yml
-rw-r--r-- 1 root root    14 Apr 22 21:51 .env

# docker-compose.yml (secret値は伏せ):
services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    container_name: npm
    restart: unless-stopped
    ports:
      - "80:80"
      - "81:81"
      - "443:443"
    volumes:
      - ${SSD_ROOT}/appdata/npm/data:/data
      - ${SSD_ROOT}/appdata/npm/letsencrypt:/etc/letsencrypt
    networks:
      - proxy-net

networks:
  proxy-net:
    external: true

# .env の KEY 一覧 (値は伏せ):
SSD_ROOT=<hidden>
```

## wireguard
```
# files:
total 16
drwxr-xr-x 2 root root  4096 Apr 23 10:51 .
drwx--S--- 9 root users 4096 Jun 26 22:14 ..
-rw-r--r-- 1 root root   661 Apr 23 10:51 docker-compose.yml
-rw-r--r-- 1 root root   123 Apr 23 10:48 .env

# docker-compose.yml (secret値は伏せ):
services:
  wireguard:
    image: lscr.io/linuxserver/wireguard:latest
    container_name: wireguard
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - PUID=1000
      - PGID=100
      - TZ=Asia/Tokyo
      - SERVERURL=${WG_HOST}
      - SERVERPORT=${WG_PORT}
      - PEERS=${WG_PEERS}
      - PEERDNS=${WG_PEERDNS}
      - INTERNAL_SUBNET=10.10.0.0
      - ALLOWEDIPS=192.168.3.0/24,10.10.0.0/24
      - LOG_CONFS=true
    volumes:
      - ${SSD_ROOT}/appdata/wireguard:/config
      - /lib/modules:/lib/modules:ro
    ports:
      - "51820:51820/udp"
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
    restart: unless-stopped

# .env の KEY 一覧 (値は伏せ):
SSD_ROOT=<hidden>
WG_HOST=<hidden>
WG_PORT=<hidden>
WG_PEERS=<hidden>
WG_INTERNAL_SUBNET=<hidden>
WG_PEERDNS=<hidden>
```

## obukata 側の compose/env ソース(newdash 方式かの判定材料)
```
-rw-r--r-- 1 obukata users 355 Jun 26 22:03 /home/obukata/cloudflared-compose.yml
-rw-r--r-- 1 obukata users 951 Jul  2 16:39 /home/obukata/newdash-compose.yml
-rw-r--r-- 1 obukata users 897 Jun 23 22:52 /home/obukata/newdash.env
```

## 稼働中の compose プロジェクト
```
NAME                STATUS              CONFIG FILES
adguardhome         running(1)          /srv/compose/adguardhome/docker-compose.yml
cloudflared         running(1)          /srv/compose/cloudflared/docker-compose.yml
immich              running(4)          /srv/compose/immich/docker-compose.yml
newdash             running(1)          /srv/compose/newdash/docker-compose.yml
nextcloud           running(3)          /srv/compose/nextcloud/docker-compose.yml,/srv/compose/nextcloud/docker-compose.override.yml
npm                 running(1)          /srv/compose/npm/docker-compose.yml
wireguard           running(1)          /srv/compose/wireguard/docker-compose.yml
```

## 各コンテナのマウント(データパス把握用)
```
-- newdash --
/proc -> /host/proc
/opt/ops/docs -> /host/docs
/srv -> /host/ssd
/srv/dev-disk-by-uuid-3a9a7624-afbb-4993-a1dd-38e2bd1546f5 -> /host/hdd

-- nextcloud --
/srv/appdata/nextcloud/html -> /var/www/html
/srv/dev-disk-by-uuid-3a9a7624-afbb-4993-a1dd-38e2bd1546f5/appdata/nextcloud/data -> /var/www/html/data

-- cloudflared --

-- immich_server --
/srv/docker/volumes/99cbf36cb6c1dcae34cb0e7f7a076c7b7b500e494780a75fdbb96e79070d959b/_data -> /data
/srv/dev-disk-by-uuid-3a9a7624-afbb-4993-a1dd-38e2bd1546f5/appdata/immich -> /usr/src/app/upload
/etc/localtime -> /etc/localtime

-- immich_machine_learning --
/srv/docker/volumes/immich_model-cache/_data -> /cache

-- immich_postgres --
/srv/appdata/immich-db -> /var/lib/postgresql/data

-- immich_redis --
/srv/docker/volumes/10cfbedd94bf6cac9ce246175bea3b854248afe7ef4dcfeb80da796b556bee7e/_data -> /data

-- wireguard --
/srv/appdata/wireguard -> /config
/lib/modules -> /lib/modules

-- npm --
/srv/appdata/npm/data -> /data
/srv/appdata/npm/letsencrypt -> /etc/letsencrypt

-- adguardhome --
/srv/appdata/adguardhome/conf -> /opt/adguardhome/conf
/srv/appdata/adguardhome/work -> /opt/adguardhome/work

-- nextcloud-db --
/srv/appdata/mariadb -> /var/lib/mysql

-- nextcloud-redis --
/srv/appdata/redis -> /data

```
