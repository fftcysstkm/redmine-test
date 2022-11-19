- [Dockerコンテナで構築したRedmineの概要](#dockerコンテナで構築したredmineの概要)
- [Redmine環境をDockerコンテナで構築する](#redmine環境をdockerコンテナで構築する)
- [サブURLでRedmineにアクセス可能にする](#サブurlでredmineにアクセス可能にする)
  - [Redmineコンテナがサブディレクトリで動作する設定をする](#redmineコンテナがサブディレクトリで動作する設定をする)
- [nginxのリバースプロキシでリクエストを振り分ける設定をする](#nginxのリバースプロキシでリクエストを振り分ける設定をする)
- [バックアップのために必要なデータをホストとマウントする](#バックアップのために必要なデータをホストとマウントする)
    - [Redmineの添付ファイル(files/)をホストとマウントする](#redmineの添付ファイルfilesをホストとマウントする)
    - [RedmineのMySQL格納データをマウントディレクトリにダンプする](#redmineのmysql格納データをマウントディレクトリにダンプする)
- [その他必要な設定](#その他必要な設定)
  - [MySQLの文字コード、タイムゾーン設定する](#mysqlの文字コードタイムゾーン設定する)
- [バックアップファイルからRedmineをリストアする](#バックアップファイルからredmineをリストアする)
- [今後のTODO](#今後のtodo)
# Dockerコンテナで構築したRedmineの概要
- RedmineのDokcerコンテナをローカルに構築した（会社の拠点間VPN内で動かす想定)。
  - nginx 1.23.2(リバースプロキシ)
  - Redmine 4.2.8(本体)
  - MySQL 5.7(Redmineのデータ格納)
  - Alpine Linux 3.16.3(MySQL定期ダンプ用、BusyBox crondを使用)
- RedmineにはサブURLでアクセス可能にした（例：`http://localhost/redmine/`。`http://localhost/`ではない。）
- nginxでリバースプロキシの設定をし、今後バックエンドのアプリ（バーチャルホスト）が増えても1つのnginxでリクエストを振り分けられるようにした。
- バックアップはDBダンプファイルとRedmineへのアップロードファイルをホストとのマウントディレクトリに格納した。
  - MySQLダンプファイル（定期ダンプ専用コンテナ:Alpine Linux）
  - Redmineコンテナのfilesディレクト配下（Redmineデフォルト）
- その他、MySQLの文字コードの設定とRedmineのリストアの方法を調査した。
- 今後のやることは、古いMySQLダンプファイルを削除するcronの設定と、Redmineの`/files`のバックアップ。

# Redmine環境をDockerコンテナで構築する
Redmine環境を、1.nginx、2.Redmine、3.MySQL、4.SQLダンプ用のDokcerコンテナで構築した。まず、nginxでクライアントからリクエストを受付ける。nginxはバックエンドのRedmineアプリに通信をリバースプロキシで振り分ける（今後、他のアプリもバーチャルホストに追加可能にするため）。RedmineはチケットをMySQLに格納する。MySQLのダンプは、ダンプ専用のコンテナで行う。ダンプ用のコンテナを作成したのは、MySQLのコンテナ内で通常のcronを実行させようとしたが、うまく動作しなかったため。<br>
docker-compose.yml。
```yaml
version: '3.1'

services:

  nginx:
    image: nginx:1.23.2
    ports:
      - 80:80
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
    depends_on:
      - redmine

  redmine:
    build:
      context: ./redmine
      dockerfile: Dockerfile-redmine
    image: redmine-4.2.8
    container_name: redmine-4.2.8
    restart: always
    volumes:
      - ./redmine/plugins:/usr/src/redmine/plugins
      - ./redmine/backup:/usr/src/redmine/files
    environment:
      REDMINE_DB_MYSQL: db
      REDMINE_DB_PASSWORD: redmine
    depends_on:
      - db

  db:
    build:
      context: ./mysql
      dockerfile: Dockerfile-mysql
    container_name: redmine-db
    ports:
      - 3306:3306
    platform: linux/amd64
    restart: always
    volumes:
      - ./mysql/backup:/var/backup
    command: mysqld --character-set-server=utf8 --collation-server=utf8_unicode_ci
    environment:
      MYSQL_ROOT_PASSWORD: redmine
      MYSQL_DATABASE: redmine
      TZ: Asia/Tokyo

  periodic-backup:
    build:
      context: ./periodic-backup
      dockerfile: Dockerfile-dbbackup-alpine
    image: alpine-for-dbbackup
    container_name: alpine-for-dbbackup
    platform: linux/amd64
    environment:
      MYSQL_CONTAINER_NAME: redmine-db
      MYSQL_DATABASE: redmine
      MYSQL_ROOT_PASSWORD: redmine
    volumes:
      - ./periodic-backup/backup:/opt/mysql/backup
    command: crond -f -d 8
    restart: always
    depends_on:
      - db
```

# サブURLでRedmineにアクセス可能にする
Redmineのアクセス時URLは、`http:/ホスト名/サブディレクトリ/`となるように、Redmineコンテナとnginxのリバースプロキシ設定をした。今回の例では、`http://localhost/redmine/`でRedmineにアクセスできる。この設定を行わないと、CSSやJavaScriptがロードされなかったり、各種リンクのURLから`/サブディレクトリ/`（今回なら`/redmine/`）が削除され404になってしまったりする。
## Redmineコンテナがサブディレクトリで動作する設定をする
RedmineがサブディレクトリでHTTP送受信をできるように、Railsの環境変数と設定ファイルのconfig.ruを編集する。
手順はRedmine[公式](https://www.redmine.org/projects/redmine/wiki/HowTo_Install_Redmine_in_a_sub-URI)を考にした。
1. Redmineのコンテナにコンテキストパスを設定する。Railsには`ENV[“RAILS_RELATIVE_URL_ROOT”] `という環境変数があり、ここにパスを設定するとコンテキストパスを変更できる模様。<br>
RedmineのDockerfile抜粋:
   ```Dockerfile
   ENV RAILS_RELATIVE_URL_ROOT=/redmine
   ```
2. Redmineのインストールディレクトリにある`config.ru`を下記のように変更する。config.ruはRackという、WebサーバとRailsアプリ間でHTTPの送受信を担当するモジュールらしい。
<br>config.ru:
   ```ruby
    require ::File.expand_path('../config/environment',  __FILE__)

    map ENV['RAILS_RELATIVE_URL_ROOT'] || '/' do
        run Rails.application
    end
   ```
# nginxのリバースプロキシでリクエストを振り分ける設定をする
nginxにリバースプロキシの設定をし、今後バックエンドのアプリ（バーチャルホスト）が増えても1つのnginxでリクエストを振り分けられるようにした。nginxの設定ファイルの意味だが、nginxは80番ポートでリクエストを待ち受け、リクエストヘッダのHostが`localhost`に合致する場合、それ以降のパスが`/redmine/`に合致するか判定する。`/redmine/`に合致した場合、`upstream`ディレクティブに定義したバックエンドのアプリにリクエストを振り分ける。`upstream`に記述した`server`には、IPアドレスの代わりにコンテナのサービス名を記述できる模様。
```properties
upstream redmine_app {
    server redmine:3000;
}

server {
    listen 80;
    server_name localhost;

    location /redmine/ {
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $hostname;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_pass http://redmine_app;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

# バックアップのために必要なデータをホストとマウントする
### Redmineの添付ファイル(files/)をホストとマウントする
現状はRedmineの`/files`ディレクトリをホストのディレクトリとマウントしているだけ(`docker-compose.yml`のRedmineコンテナの`vlumes`参照)。TODOとして3日分くらいバックアップして、古いファイルは削除する。
### RedmineのMySQL格納データをマウントディレクトリにダンプする
Alpine LinuxベースのMySQLダンプ専用のコンテナを作り、毎日1回ダンプファイルを圧縮し、マウントディレクトリに格納する。Alpine Linuxのコンテナイメージには、MySQLをダンプ・圧縮するシェルスクリプトをホストからコピーしておく。また、タイムゾーンを日本にし、MySQL接続クライアントをインストールする。[こちら](https://ricardolsmendes.medium.com/mysql-mariadb-with-scheduled-backup-jobs-running-in-docker-1956e9892e78)参考にさせてもらった。当初MySQLコンテナ内でcronを実行しようとしていたが、なぜかうまく動作せず、このブログにたどり着いた。ブログによれば、1コンテナ1ソフトウェアの原則というものがあるようで、たしかに独立させると後で気軽にダンプの設定変更しやすいと思った。<br>
Dcokerfile-dbbackup-alpine:
```dockerfile
FROM alpine:3.16.3

COPY ./scripts/daily/* /etc/periodic/daily

# 時刻をJSTにする。MySQL接続用クライアントをインストールする
RUN apk update && \
    apk upgrade && \
    apk --no-cache add tzdata && \
    cp /usr/share/zoneinfo/Asia/Tokyo /etc/localtime && \
    apk del tzdata && \
    apk add --no-cache mysql-client && \
    chmod a+x /etc/periodic/daily/*
```
ダンプ・圧縮するシェルスクリプト:
```sh
#!/bin/sh

BACKUP_FOLDER=/opt/mysql/backup
NOW=$(date '+%Y%m%d%H%M')

GZIP=$(which gzip)
MYSQLDUMP=$(which mysqldump)

### MySQL Server Login info ###
MDB=redmine
MHOST=db
MPASS=redmine
MUSER=root

[ ! -d "$BACKUP_FOLDER" ] && mkdir --parents $BACKUP_FOLDER

FILE=${BACKUP_FOLDER}/backup-${NOW}.sql.gz
$MYSQLDUMP -h $MHOST -u $MUSER -p${MPASS} --databases $MDB | $GZIP -9 > $FILE
```
# その他必要な設定
## MySQLの文字コード、タイムゾーン設定する
そのままです。<br>
`docker-compose.yml`一部再掲:
```yml
  db:
    build:
      context: ./mysql
      dockerfile: Dockerfile-mysql
    container_name: redmine-db
    ports:
      - 3306:3306
    platform: linux/amd64
    restart: always
    volumes:
      - ./mysql/backup:/var/backup
    command: mysqld --character-set-server=utf8 --collation-server=utf8_unicode_ci
    environment:
      MYSQL_ROOT_PASSWORD: redmine
      MYSQL_DATABASE: redmine
      TZ: Asia/Tokyo
```
# バックアップファイルからRedmineをリストアする
MySQLのコンテナに入り、解答したダンプファイルを下記コマンドで読み込ませる。
`mysql -u MySQLユーザー名 -pMySQLパスワード -hホスト名 Redmineデータベース名 < ダンプデータファイル名`<br>
今回の場合:<br>
`mysql -u root -predmine -hdb redmine < /var/backup/backup_redmine20221112.sql`

# 今後のTODO
ファイルが増え続けないよう、定期的にSQLダンプファイルを削除するcronが必要。また、Redmineの`/files`配下も同様にバックアップする必要あり。
