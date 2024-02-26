---
title: "BlueskyのPDSを建てる (0.3.0-beta)"
emoji: "🦋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [bluesky]
published: true
---

> **2024年2月26日追記**
> 本記事は少し古くなっています。新しいPDSのバージョンのインストール手順と、Blueskyの連合に参加する方法について別記事に記載しました
> 👉 [PDSを建ててBlueskyの連合に参加する (0.4.0-beta)](ed1448e8ed5257)



度々の話題になっているBlueskyに招待してもらい、約9ヵ月ほど。
自身で関連アプリを作ったりもしたが、いまさらになってPDSを建ててみたのでそのメモを残しておきます。

以前よりもドキュメントや初期設定も整備され、建てるだけならすぐにできるようになっています。

# サーバーを準備

手頃なサーバーを準備します。何でも良いですが、公式のドキュメントに沿うならOSはUbuntu 22.xを使用します。
またこのスペックを求めると ~~Raspberry Pi 4での運用は厳しいかもしれない~~ Raspberry Pi 5なら満たせそうです。


| 項目 | 性能 |
| ---- | ------------------- |
| CPU  | 2 vCPU     |
| メモリ  | 2 GB       |
| ディスク | SSD 200 GB |

#### 所感

お一人様運用あるいはフォローの少ないほぼ初期状態で動かしている場合、最低でも以下のスペックでもいけるかも。
サンドボックスで検証環境として動かすだけなら良いかもしれません。これはサンドボックス環境で一つだけアカウントがある実行時からみたスペックです。

| 項目 | 性能 |
| ---- | ------------------- |
| CPU  | 2 vCPU              |
| メモリ  | 1 GB  (最低でも 512 MB) |
| ディスク | 20 GB               |

## SMTPサーバー

SMTPは自前で持ちたくないので [Mailgun](https://mailgun.com) を使用します。

Gmail、[Mailtrap](https://mailtrap.io)、[MailerSend](https://www.mailersend.com/)、[Sendgrid](https://sendgrid.kke.co.jp)、[AWS SES](https://aws.amazon.com/jp/ses/)（無料枠が1年のみ、制限が厳しくなった、従量）でもSMTPが使えれば何でもいいです。


## データベース

デフォルトだとSQLiteで実行しますが、今回はPostgreSQLを使用します。

## DNS

結構重要になります。 今回は[Cloudflare](https://cloudflare.com)を使用しました。レコード設定は、以下記載の分を準備します。

### PDS用

- PDSを動かすための主ドメイン `bsky.social` 相当。
- [/xrpc/com.atproto.server.describeServer](https://bsky.social/xrpc/com.atproto.server.describeServer) などのAPIが動作できるようにする必要があります。

### ハンドル用

- **ハンドル用にとして使用しますが、DNSで名前解決できる必要があります**。そのため、ワイルドカードで受け付けられる必要があります。
- `*.bsky.soial`、`*.bsky.name` など。
- ちなみに、上記のPDS用 と同一にする必要はないようです。
  - イレギュラーですが、例えばPDSを `mypds.example` 、ハンドルを `.handle.example` とすることも、仕様上は可能です。

### Cloudflareでの運用上の注意

Cloudflareで利用する場合、CDNを有効にした際の[無料のUniversalSSL証明書での扱いは注意が必要](https://developers.cloudflare.com/ssl/edge-certificates/additional-options/total-tls/error-messages/)です。
UniversalSSL証明書がカバーする範囲はAPEX(`example.com`)およびサブドメイン（`*.example.com`）までとなります。 `*.foo.example.com`といった2階層目以降のサブドメインにはUniversal証明書は有効になりません。

上記仕様のため、ハンドルを2階層以降のサブドメインで運用する場合、ハンドルへの問い合わせに対してはCDN経由で有効にできない可能性があります。

あるいは、有償のAdvanced、または証明書持ち込みのCustomを選択する事で解決できるかもしれません。

# インストール

基本的に[ドキュメント](https://github.com/bluesky-social/pds/blob/main/README.md)に沿って、コマンドラインからコマンド実行で動作環境は作成できます。

[ドキュメント内にもある](https://github.com/bluesky-social/pds/blob/main/README.md#automatic-install-on-ubuntu-20042204-or-debian-1112)ように、以下のようにインストーラー(`./installer.sh`)を利用して実行するとスムーズです。

```yaml
wget https://raw.githubusercontent.com/bluesky-social/pds/main/installer.sh
```

または

```yaml
git clone https://github.com/bluesky-social/pds
cd pds
sudo bash ./installer.sh
```

途中、`$PDS_ADMIN_PASSWORD` を生成しますが、これは管理APIを使用する際（管理者権限による招待コードの発行など）に必要なので管理、取り扱いに注意が必要です。

## PDSのエントリポイントホスト名を設定する

PDSのホスト名を指定します。またPDS内の各ハンドルがどのPDSに所属しているかの問い合わせ先として使用されます。

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

pds.envでDBを指定します。SQLiteが指定されている行を削除またはコメントアウト。
**PGSQLとSQLiteの設定が同居している場合、同担不可の警告が表示されます**。

```shell
#PDS_DB_SQLITE_LOCATION=/pds/pds.sqlite
```

PostgreSQLを指定します。接続先は `network_mode: host` で実行してる場合は`localhost`を指定すればOK。

```shell
PDS_DB_POSTGRES_URL=postgres://postgres:pgpassword@localhost/pds
```

## メール送信に対応する

`pds.env` をアップデートし、SMTPサーバーの指定を追加します

```shell
SMTP_USERNAME=smtp-user
SMTP_PASSWORD=smtp-password
```

さらに、`PDS_EMAIL_SMTP_URL` にSMTPサーバーへの接続をURIで記述します。下記の例では[Mailgun](https://mailgun.com)を使用しています。ほかのプロバイダを使用する場合は適宜読み替えてください。

```shell
PDS_EMAIL_SMTP_URL=smtps://${SMTP_USERNAME}:${SMTP_PASSWORD}@smtp.mailgun.org
```

なお、Mailgunではユーザー名に `user@example.com` 形式のフォーマットを使用するため、`$SMTP_USERNAME` に指定するユーザー名は `@` を `%40` に置換して記述する必要があります。

例 `user%40example.com`


#### PDS 0.3.0-beta.3 ではメールアドレス検証は利用できない

ただし、PDS 0.3.0-beta3 ではまだメールアドレス検証のメソッドが実装されていません。

利用中のPDSのバージョンは [/xrpc/_health](https://bsky.social/xrpc/_health) で確認できます。

# カスタマイズ

## システム予約済みハンドルを取得する

この項目はおまけなので分からない方は避けてください。
システムで予約されているハンドルが結構あります。これは過去に著名な組織や公式の名を騙ったハンドルを取得し、売買しようとした荒らしによる影響があります。

自分のハンドルも **anon** なのですが、 Anonymous Trolling 対策として予約されています。

これらはコード内にハードコードされているため、基本的に無効化できません。
ただし、PDSを実行しているDockerコンテナ内でに入り、当該部分を直接編集する事でこれを回避する事ができます。

例えば、以下の手順です。

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

先述の通り、PDSのドメイン `$PDS_HOSTNAME` と同一である必要はありません。
また、仕様上は複数のデフォルトサフィックスを設定できるようにもなっています。

```shell
PDS_SERVICE_HANDLE_DOMAINS=".example.test,.foo.test,.bar.test,.baz.tes,.qux.test"
```

ただし[Blueskyアプリ](https://bsky.app)を含め、ユーザー登録画面や登録処理では現在のところ先頭のサフィックスドメインが使用され、残りのものについては選択、設定するUIもないため事実上設定できなくなっています。

## 利用規約、プライバシーポリシーを設置する

各PDSでそれぞれ利用規約、プライバシーポリシーを掲げる場合、以下のパラメータに設定すると、新規アカウント登録ページでリンクが表示されます。

```shell
PDS_TERMS_OF_SERVICE_URL=https://blueskyweb.xyz/support/tos
PDS_PRIVACY_POLICY_URL=https://blueskyweb.xyz/support/privacy-policy
```

![ScreenShot 2024-01-24 12.47.43.png](https://res.craft.do/user/full/9289c338-8841-272e-ba00-54f103dbb7a5/doc/2084B0DB-186A-4ED7-B1CC-DAE711EC85F1/CBC72CAE-33CE-4BE2-80BD-903FBFD74C4C_2/xjA225xpySyottG2ClToDaYeRdl4j7zCzHhxmjvsFhYz/ScreenShot%202024-01-24%2012.47.43.png)

## 招待コードの有無、発行スケジュールを調整する

PDS登録時に招待コードの必要可否を設定するには以下のフラグで設定します。デフォルトはこの項目がなくても招待コードが必要になっています。

```shell
PDS_INVITE_REQUIRED=true
```

`false` にすると招待コード無しで登録可能になる。

招待コードの発行間隔は以下のパラメータで設定できます。
値の単位はミリ秒。デフォルトは1週間(`604800000`)になっています。

```shell
PDS_INVITE_INTERVAL=604800000
```
例えば
- 毎日1つ発行したい場合 1 day = 24 hrs = 86400 sec =  `86400000` msec
- bsky.social と同じにする場合 10 days = 240 hrs = 864000 sec = `864000000` msec

## PDSの状態を確認する

`/xrpc/com.atproto.server.describeServer` にGETでリクエストを送信すると、設定されているPDSの情報がJSONで取得できます。

----
# 更新履歴

- 初稿でタイトルに (v1.3.3) と入れていましたが、異なるバージョン情報だったため、PDSのバージョンとして変更しました。(2024/02/25)
- メモ程度の内容だったため雑だった文体をですます調に修正しました (2024/02/25)
- Blueskyの連合開始時のPDS作成記事へのリンクを追加しました (2024/02/26)
