---
title: "BlueskyのPDSを建てる (0.3.0-beta)"
emoji: "🦋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [bluesky]
published: true
---

度々の話題になっているBlueskyに招待してもらい、約9ヵ月ほど。
自身で関連アプリを作ったりもしたが、いまさらになってPDSを建ててみたのでそのメモを残す。

以前よりもドキュメントや初期設定も整備され、建てるだけならすぐにできるようになっている。

# サーバーを準備

手頃なサーバーを準備。何でも良いが、公式のドキュメントに沿うならOSはUbuntu 22.x。
また、このスペックを求めるとRaspberry Piでの運用は厳しい。


| 項目 | 性能 |
| ---- | ------------------- |
| CPU  | 2 vCPU     |
| メモリ  | 2 GB       |
| ディスク | SSD 200 GB |

#### 所感

お一人様運用、フォローの少ないほぼ初期状態で動かしている場合、最低でも以下のスペックでもいけそう。サンドボックスで検証環境として動かすだけなら良さそう。

| 項目 | 性能 |
| ---- | ------------------- |
| CPU  | 2 vCPU              |
| メモリ  | 1 GB  (最低でも 512 MB) |
| ディスク | 20 GB               |

## SMTPサーバー

SMTPは自前で持ちたくないので [Mailgun](https://mailgun.com) を使用。

Gmail、[Mailtrap](https://mailtrap.io)、[MailerSend](https://www.mailersend.com/)、[Sendgrid](https://sendgrid.kke.co.jp)、[AWS SES](https://aws.amazon.com/jp/ses/)（無料枠が1年のみ、制限が厳しくなった、従量）でもSMTPが使えれば何でも良い。

## データベース

デフォルトだとSQLiteで動作。今回はPostgreSQLを使用。

## DNS

結構重要。 今回は[Cloudflare](https://cloudflare.com)を使用。レコード設定は以下の分を準備。

### PDS用

- PDSを動かすための主ドメイン bsky.social相当。
- [/xrpc/com.atproto.server.describeServer](https://bsky.social/xrpc/com.atproto.server.describeServer) などのAPIが動作できるようにする

### ハンドル用

- ::ハンドル用に名前解決できる必要がある::。ワイルドカードで受け付けられる必要がある。
- PDS用と同一にする必要はない。
- `*.bsky.soial`、`*.bsky.name` など。

### Cloudflareでの運用上の注意

Cloudfareで利用する場合、CDNを有効にした際の[無料のUniversalSSL証明書での扱いは注意が必要](https://developers.cloudflare.com/ssl/edge-certificates/additional-options/total-tls/error-messages/)である。
UniversalSSL証明書がカバーする範囲はAPEX(`example.com`)およびサブドメイン（`*.example.com`）までである。 `*.foo.example.com`といった2階層目。以降のサブドメインには有効にならない。

上記によりハンドルをサブドメイン以降で運用する場合、ハンドルへの問い合わせに対してはCDN経由で有効にできない可能性があることに注意する。

# インストール

基本的に[ドキュメント](https://github.com/bluesky-social/pds/blob/main/README.md)に沿ってコピペ実行で動作環境は作成できる。

[ドキュメント内にもある](https://github.com/bluesky-social/pds/blob/main/README.md#automatic-install-on-ubuntu-20042204-or-debian-1112)ように、以下のようにインストーラーを利用して実行することもできる。

```yaml
wget https://raw.githubusercontent.com/bluesky-social/pds/main/installer.sh
```

または

```yaml
git clone https://github.com/bluesky-social/pds
cd pds
sudo bash ./installer.sh
```

途中、`$PDS_ADMIN_PASSWORD` を生成するが、これは管理APIを使用する際（管理者権限による招待コードの発行など）に必要なので管理、取り扱いに注意すること。

## PDSのエントリポイントホスト名を設定する

PDSのホスト名。アカウント作成時に指定する。また各ハンドルのPDS問い合わせ先として使用される。

```shell
PDS_HOSTNAME=entry.example.local
```

## データベースをPostgreSQLに変更する

### PostgreSQL用初期設定

```shell
mkdir -p /pds/postgres/data
mkdir -p /pds/postgres/init
cat <<_INIT_QUERY_ > /pds/postgres/init/init.sql
CREATE DATABASE pds;
GRANT ALL PRIVILEGES ON DATABASE pds TO postgres;
_INIT_QUERY_
```

### 環境変数

`/pds/postgres.env` に以下を設定

```shell
POSTGRES_INITDB_ARGS=--encoding=UTF-8 --lc-collate=C --lc-ctype=C
POSTGRES_USER=postgres
POSTGRES_PASSWORD=pgpassword
POSTGRES_DB=healthcheck
```

### compose.yml のアップデート

```yaml
version: '3.9'
services:
# ...
  postgres:
    container_name: database
    network_mode: host
    hostname: pgdb
    image: postgres:16-alpine
    restart: always
    env_file:
      - /pds/postgres.env
    volumes:
      - /pds/postgres/init/:/docker-entrypoint-initdb.d/
      - /pds/postgres/database/:/var/lib/postgresql/data/
    healthcheck:
      test: "pg_isready -U postgres -d postgres"
      interval: 5s
      retries: 20
```

pds.envでDBを指定する。SQLiteが指定されている行を削除またはコメントアウト。::PGSQLとSQLiteの設定が同居している場合、同担不可の警告が表示される::。

```shell
#PDS_DB_SQLITE_LOCATION=/pds/pds.sqlite
```

PostgreSQLを指定。接続先は `network_mode: host` で実行してる場合は`localhost`を指定。

```shell
PDS_DB_POSTGRES_URL=postgres://postgres:pgpassword@localhost/pds
```

## メール送信に対応する

`pds.env` をアップデートし、SMTPサーバーの指定を追加する

```shell
SMTP_USERNAME=smtp-user
  SMTP_PASSWORD=smtp-password
```

さらに、`PDS_EMAIL_SMTP_URL` にSMTPサーバーへの接続をURIで記述する。下記の例では[Mailgun](https://mailgun.com)を使用している。

```shell
PDS_EMAIL_SMTP_URL=smtps://$SMTP_USERNAME:$SMTP_PASSWORD@smtp.mailgun.org
```

なお、Mailgunではユーザー名に `user@example.com` 形式のフォーマットを使用するため、`$SMTP_USERNAME` に指定する際は `@` を `%40` に置換して記述すること。 例 `user%40example.com`

#### PDS 0.3.0-beta.3 ではメールアドレス検証は利用できない

PDSのバージョンは [/xrpc/_health](https://bsky.social/xrpc/_health) で確認できる。

# カスタマイズ

## システム予約済みハンドルを取得する

システムで予約されているハンドルが結構ある。過去の荒らしによる影響。

自分のハンドルも **anon** なので Anonymous Trolling 対策で予約されている。

Dockerコンテナ内では以下の手順で取得できるようにできる。

1. コンテナにログインする

```shell
docker exec -it pds sh
```

1. `/app/node_modules/@atproto/pds/dist/index.js` を `vi` で開く
2. reservedHandle 一覧があるので検索
3. 利用したいハンドルをコメントアウト
4. コンテナを再起動

```shell
docker compose restart pds
```

## PDSのハンドルで利用可能なサフィックスドメインを複数指定する

`$PDS_HOSTNAME` と同一である必要はない。また、仕様上は複数のデフォルトサフィックスを設定できる。

```shell
PDS_SERVICE_HANDLE_DOMAINS=".example.test,.foo.test,.bar.test,.baz.tes,.qux.test"
```

ただし[Blueskyアプリ](https://bsky.app)を含め、ユーザー登録画面や登録処理では現在のところ先頭のサフィックスドメインが使用され、残りのものについては選択することができない。

## 利用規約、プライバシーポリシーを設置する

別途利用規約、プライバシーポリシーを掲げる場合、以下のパラメータに設定すると、新規アカウント登録ページでリンクが表示される。

```shell
PDS_TERMS_OF_SERVICE_URL=https://blueskyweb.xyz/support/tos
PDS_PRIVACY_POLICY_URL=https://blueskyweb.xyz/support/privacy-policy
```

![ScreenShot 2024-01-24 12.47.43.png](https://res.craft.do/user/full/9289c338-8841-272e-ba00-54f103dbb7a5/doc/2084B0DB-186A-4ED7-B1CC-DAE711EC85F1/CBC72CAE-33CE-4BE2-80BD-903FBFD74C4C_2/xjA225xpySyottG2ClToDaYeRdl4j7zCzHhxmjvsFhYz/ScreenShot%202024-01-24%2012.47.43.png)

## 招待コードの有無、発行スケジュールを調整する

PDS登録時に招待コードの有無を設定するのは以下のフラグ。デフォルトは有効。

```shell
PDS_INVITE_REQUIRED=true
```

`false` にすると招待コード無しで登録可能。

招待コードの発行間隔は以下のパラメータで設定。値はミリ秒。デフォルトは1週間(`604800000`)。

```shell
PDS_INVITE_INTERVAL=604800000
```

- 毎日1つ発行したい場合 1 day = 24 hrs = 86400 sec =  `86400000` msec
- bsky.social と同じにする場合 10 days = 240 hrs = 864000 sec = `864000000` msec

## PDSの公開されている設定状態を確認する

`/xrpc/com.atproto.server.describeServer` にGETでリクエストを投げると設定されている情報が表示される

