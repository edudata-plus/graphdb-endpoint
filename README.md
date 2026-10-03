# rdf.edudata.jp SPARQL サービス

`rdf.edudata.jp` で公開する SPARQL サービスの構成・設定・運用方法をまとめる。

本サービスでは、以下の2つのデータセットを GraphDB で提供する。

- Textbook LOD (`jp-textbook`)
- Course of Study LOD (`jp-cos`)

一般利用者には SPARQL endpoint と SPARQL UI のみを公開し、GraphDB Workbench および管理機能は外部公開しない。

## 構成

```text
Internet
   |
   | HTTPS
   v
Apache2
rdf.edudata.jp
   |
   +-- /sparql/
   |      |
   |      +-- MatGUI / Yasgui
   |
   +-- /sparql-endpoint/textbook
   |      |
   |      +-- http://127.0.0.1:7200/repositories/jp-textbook
   |
   +-- /sparql-endpoint/cos
          |
          +-- http://127.0.0.1:7200/repositories/jp-cos

GraphDB
  Docker container
  127.0.0.1:7200 のみで待ち受け
```

公開 URL:

```text
https://rdf.edudata.jp/sparql/

https://rdf.edudata.jp/sparql-endpoint/textbook
https://rdf.edudata.jp/sparql-endpoint/cos
```

GraphDB Workbench:

```text
http://127.0.0.1:7200/
```

Workbench は localhost からのみアクセス可能とする。

---

# 1. GraphDB

## 1.1 Docker Compose

GraphDB は Docker で起動する。

例:

```yaml
services:
  graphdb:
    image: ontotext/graphdb:11.5.1
    container_name: graphdb

    restart: unless-stopped

    ports:
      - "127.0.0.1:7200:7200"

    volumes:
      - ./graphdb-home:/opt/graphdb/home
```

重要なのは以下の指定である。

```yaml
ports:
  - "127.0.0.1:7200:7200"
```

これにより GraphDB の 7200 番ポートは外部公開されず、ホスト OS 上の Apache からのみアクセスできる。

## 1.2 起動

```bash
docker compose up -d
```

状態確認:

```bash
docker compose ps
```

ログ確認:

```bash
docker compose logs -f graphdb
```

停止:

```bash
docker compose down
```

再起動:

```bash
docker compose restart
```

## 1.3 サーバ再起動時の自動起動

Compose に

```yaml
restart: unless-stopped
```

を指定する。

設定確認:

```bash
docker inspect -f '{{.HostConfig.RestartPolicy.Name}}' graphdb
```

以下が返ればよい。

```text
unless-stopped
```

Docker 自体が自動起動することも確認する。

```bash
systemctl is-enabled docker
```

必要なら:

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

---

# 2. GraphDB の repository

GraphDB 上に以下の repository を作成する。

```text
jp-textbook
jp-cos
```

SPARQL endpoint はそれぞれ以下になる。

```text
http://127.0.0.1:7200/repositories/jp-textbook
http://127.0.0.1:7200/repositories/jp-cos
```

---

# 3. GraphDB のアクセス制御

GraphDB の Security を有効にする。

Workbench の:

```text
Setup
→ Users and Access
→ Security
```

から Security を ON にする。

管理者アカウントには管理権限を設定する。

匿名アクセス用の Free Access user には、対象 repository の Read 権限のみを与える。

```text
jp-textbook
  Read:  yes
  Write: no

jp-cos
  Read:  yes
  Write: no
```

これにより匿名利用者は SPARQL query を実行できるが、SPARQL Update や repository 管理はできない。

Apache 側でも GraphDB Workbench や管理 API は proxy しない。

---

# 4. Apache2

## 4.1 必要なモジュール

```bash
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod ssl
```

必要に応じて:

```bash
sudo systemctl reload apache2
```

## 4.2 VirtualHost

設定ファイル例:

```text
/etc/apache2/sites-available/rdf.edudata.jp.conf
```

### HTTP

```apache
<VirtualHost *:80>
    ServerName rdf.edudata.jp

    Redirect permanent / https://rdf.edudata.jp/
</VirtualHost>
```

### HTTPS

```apache
<VirtualHost *:443>
    ServerName rdf.edudata.jp

    SSLEngine on

    SSLCertificateFile /etc/letsencrypt/live/rdf.edudata.jp/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/rdf.edudata.jp/privkey.pem

    # SPARQL UI
    Alias /sparql/ /var/www/yasgui/

    <Directory /var/www/yasgui/>
        Require all granted
        Options -Indexes
    </Directory>

    RedirectMatch 301 ^/sparql$ /sparql/

    # Textbook LOD
    ProxyPass \
        /sparql-endpoint/textbook \
        http://127.0.0.1:7200/repositories/jp-textbook

    ProxyPassReverse \
        /sparql-endpoint/textbook \
        http://127.0.0.1:7200/repositories/jp-textbook

    # Course of Study LOD
    ProxyPass \
        /sparql-endpoint/cos \
        http://127.0.0.1:7200/repositories/jp-cos

    ProxyPassReverse \
        /sparql-endpoint/cos \
        http://127.0.0.1:7200/repositories/jp-cos

    ProxyRequests Off

    ErrorLog ${APACHE_LOG_DIR}/rdf.edudata.jp-error.log
    CustomLog ${APACHE_LOG_DIR}/rdf.edudata.jp-access.log combined
</VirtualHost>
```

設定を有効化する。

```bash
sudo a2ensite rdf.edudata.jp.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

VirtualHost の確認:

```bash
sudo apache2ctl -S
```

---

# 5. HTTPS / Let's Encrypt

Certbot と Apache plugin をインストールする。

Ubuntu / Debian:

```bash
sudo apt update
sudo apt install certbot python3-certbot-apache
```

証明書取得:

```bash
sudo certbot --apache -d rdf.edudata.jp
```

証明書確認:

```bash
sudo certbot certificates
```

更新テスト:

```bash
sudo certbot renew --dry-run
```

systemd timer の確認:

```bash
systemctl status certbot.timer
```

証明書は通常以下に置かれる。

```text
/etc/letsencrypt/live/rdf.edudata.jp/fullchain.pem
/etc/letsencrypt/live/rdf.edudata.jp/privkey.pem
```

---

# 6. SPARQL UI

SPARQL UI は MatGUI 版 Yasgui を利用する。

配置先:

```text
/var/www/yasgui/index.html
```

---

# 7. Endpoint の切り替え

MatGUI の `endpointButtons` を使い、UI 上で以下の endpoint を切り替えられるようにする。

```text
Textbook LOD
Course of Study LOD
```

それぞれ:

```text
/sparql-endpoint/textbook
/sparql-endpoint/cos
```

に対応する。

---

# 8. リンク元からデフォルト endpoint を指定する

MatGUI 標準の `endpoint` URL parameter を利用する。

Textbook LOD から SPARQL UI へリンクする場合:

```text
https://rdf.edudata.jp/sparql/?endpoint=%2Fsparql-endpoint%2Ftextbook
```

Course of Study LOD からリンクする場合:

```text
https://rdf.edudata.jp/sparql/?endpoint=%2Fsparql-endpoint%2Fcos
```

MatGUI 側では:

```javascript
populateFromUrl: true
```

としておく。

独自に `dataset=jp-cos` などを解釈して endpoint を後から変更する方法は、MatGUI の localStorage によるタブ復元処理と競合することがあるため使用しない。

---

# 9. localStorage について

MatGUI / Yasgui はタブや endpoint の状態をブラウザの localStorage に保存する。

このため、以前にアクセスしたことがあるブラウザでは、直前にアクセスした endpoint やクエリ内容等が復元されることがある。

前節でのリンク元からの移行で混乱を来たさないよう、MatGUI の persistence を無効化することにより、この状態保存機能を無効化した。

---

# 10. Font Awesome

MatGUI の CSS には Font Awesome が含まれているが、npm package に対応する `.woff2` ファイルが含まれていないバージョンがある。

そのため、Font Awesome を別途 CDN から読み込んでいる。

```html
<link
  rel="stylesheet"
  href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/7.3.1/css/all.min.css"
>
```

MatGUI CSS より後に読み込む。

将来的には MatGUI の配布パッケージ側で修正された場合、この回避策を再確認する。

---

# 11. 動作確認

## HTTP → HTTPS

```bash
curl -I http://rdf.edudata.jp/
```

以下のような応答を期待する。

```text
HTTP/1.1 301 Moved Permanently
Location: https://rdf.edudata.jp/
```

## SPARQL UI

```bash
curl -I https://rdf.edudata.jp/sparql/
```

## Textbook LOD

```bash
curl \
  -H 'Accept: application/sparql-results+json' \
  --data-urlencode \
  'query=SELECT * WHERE { ?s ?p ?o } LIMIT 1' \
  https://rdf.edudata.jp/sparql-endpoint/textbook
```

## Course of Study LOD

```bash
curl \
  -H 'Accept: application/sparql-results+json' \
  --data-urlencode \
  'query=SELECT * WHERE { ?s ?p ?o } LIMIT 1' \
  https://rdf.edudata.jp/sparql-endpoint/cos
```

## GraphDB の localhost 接続

```bash
curl http://127.0.0.1:7200/
```

## GraphDB が外部公開されていないこと

外部ホストから:

```bash
curl http://rdf.edudata.jp:7200/
```

接続できないことを確認する。

---

# 12. ログ

GraphDB:

```bash
docker compose logs -f graphdb
```

Apache:

```text
/var/log/apache2/rdf.edudata.jp-access.log
/var/log/apache2/rdf.edudata.jp-error.log
```

Certbot:

```text
/var/log/letsencrypt/letsencrypt.log
```

---

# 13. 更新

GraphDB image の更新:

```bash
docker compose pull
docker compose up -d
```

更新後:

```bash
docker compose ps
docker compose logs graphdb
```

を確認する。

GraphDB の major/minor version を変更する場合は、repository データとの互換性を事前に確認する。

---

# 14. バックアップ

永続データは Compose の:

```yaml
volumes:
  - ./graphdb-home:/opt/graphdb/home
```

でホスト側に保存される。

したがって、少なくとも以下をバックアップ対象とする。

```text
graphdb-home/
compose.yml
/etc/apache2/sites-available/rdf.edudata.jp.conf
/var/www/yasgui/
```

GraphDB データについては、稼働中の単純なファイルコピーだけに依存せず、GraphDB の repository export / backup 方法も併用することを推奨する。

---

# 15. 運用方針

公開範囲は以下に限定する。

```text
/sparql/
/sparql-endpoint/textbook
/sparql-endpoint/cos
```

GraphDB Workbench、REST 管理 API、7200 番ポートは Internet に公開しない。
管理者は、サーバへの SSH アクセス可能なマシンから、ポートフォワード機能を使ってアクセスし、 http://localhost:7200/ にアクセスして管理機能を実行すること。

```bash
ssh -L 7200:127.0.0.1:7200 rdf.edudata.jp
```

匿名ユーザーには repository の Read 権限のみを与える。

Apache と GraphDB の双方で公開範囲を制限し、SPARQL endpoint を Read-only の公開サービスとして運用する。