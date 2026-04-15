# Docker 個人知識ベース

## Colima をセットアップする

> [!info] なぜ Colima か
> macOS では Docker Desktop の代替として使う。Docker Desktop はライセンス制限があるが、Colima は OSS で無料。VM 上で Docker デーモンを動かす仕組みは同じ。

### インストール

```bash
brew install colima
brew install docker
brew install docker-compose
brew install docker-buildx
```

### Docker 設定

`~/.docker/config.json`

```json
{
    "auths": {},
    "currentContext": "colima",
    "cliPluginsExtraDirs": [
        "/opt/homebrew/lib/docker/cli-plugins"
    ]
}
```

`cliPluginsExtraDirs` を設定することで `docker compose` コマンドが使えるようになる。
パスは Homebrew のインストール先によって異なるため、`brew --prefix` で確認すること。

### VM 操作

```bash
colima start
colima stop
```

ホスト再起動時の自動起動

```bash
brew services start colima  # 有効化
brew services stop colima   # 無効化
```

## コンテナを操作する

### 確認

```bash
docker ps     # 起動中のみ
docker ps -a  # 停止中を含む全一覧
```

### 起動・停止・削除

```bash
docker run -it --name <name> -p 8080:80 <image>  # フォアグラウンド
docker run -d  --name <name> -p 8080:80 <image>  # バックグラウンド
docker stop <container>
docker rm <container>
docker rm $(docker ps -aq)                        # 停止中を一括削除
```

### シェル・ログ・ファイル

```bash
docker exec -it <container> /bin/bash       # コンテナ内に入る
docker logs    <container>                  # ログ確認
docker logs -f <container>                  # ログをフォロー
docker cp      <container>:/path/to/file .  # ファイルをコピー
```

## 環境変数を渡す

### コマンドラインで指定

```bash
docker run -e KEY=VALUE -e ANOTHER=VALUE <image>
```

### ファイルから一括読み込み

`.env` ファイルを用意する。

```
DB_HOST=localhost
DB_PORT=5432
```

```bash
docker run --env-file .env <image>
```

### docker compose での記法

```yaml
services:
  app:
    image: myapp
    environment:
      - KEY=VALUE
    env_file:
      - .env
```

## イメージを管理する

```bash
docker images         # 一覧
docker pull <image>   # 取得
docker rmi  <image>   # 削除
docker image prune    # 未使用を削除
```

### ビルド

```bash
docker build -t <name>:<tag> .
docker build -t <name>:<tag> -f <Dockerfile path> .
```

## docker compose で複数コンテナを扱う

```bash
docker compose up -d                      # バックグラウンドで起動
docker compose down                       # 停止・コンテナ・ネットワークを削除
docker compose down -v                    # ボリュームも削除
docker compose ps                         # 状態確認
docker compose logs -f                    # ログをフォロー
docker compose exec <service> /bin/bash   # コンテナに入る
docker compose build                      # イメージをビルド
docker compose pull                       # イメージを更新
```

## データを永続化する（ボリューム）

### named volume（推奨）

```bash
docker volume create myvolume
docker run -v myvolume:/data <image>
docker volume ls
docker volume rm myvolume
```

### データボリュームコンテナ（旧パターン）

> `--volumes-from` は古いパターン。新規用途では named volume を使う。

ボリュームコンテナを作成する。

```bash
docker run --name=bb-container -v bb-volume:/shared busybox
```

別コンテナからマウントして使う。

```bash
docker run --rm --volumes-from bb-container -it ubuntu /bin/bash
```

`docker container prune` でコンテナを削除してもボリューム自体は残る。

### バックアップ・リストア

```bash
# バックアップ
docker run --rm \
  --volumes-from bb-container \
  -v $HOME/backup:/backup \
  ubuntu \
  tar cvf /backup/container-bkup.tar -C / shared

# リストア
docker run --rm \
  --volumes-from bb-container \
  -v $HOME/backup:/backup \
  ubuntu \
  tar xvf /backup/container-bkup.tar -C /
```

## 使い捨て環境として使う

> [!info] なぜ使い捨て環境か
> `--rm` でコンテナ終了時に自動削除される。ローカル環境を汚さずに任意のランタイムを試せる。バージョン違いの動作確認や、インストールしたくないツールの一時実行に使う。

```bash
# Python の即席 REPL
docker run --rm -it python:3.12 python

# ローカルスクリプトを Node.js で実行
docker run --rm -v $(pwd):/app -w /app node:20 node script.js

# Ubuntu で探索
docker run --rm -it ubuntu:24.04 /bin/bash

# Go のビルドをコンテナ内で実行
docker run --rm -v $(pwd):/work -w /work golang:1.22 go build ./...
```

## DB をすぐ立ち上げる

> データ永続化が必要な場合は `-v` でボリュームをマウントし、`--rm` を外す。

### PostgreSQL

```bash
docker run --rm --name pg \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=mydb \
  -p 5432:5432 \
  postgres:16
```

### Redis

```bash
docker run --rm --name redis \
  -p 6379:6379 \
  redis:7
```

### MySQL

```bash
docker run --rm --name mysql \
  -e MYSQL_ROOT_PASSWORD=password \
  -e MYSQL_DATABASE=mydb \
  -p 3306:3306 \
  mysql:8
```

## 調査・デバッグする

コンテナの詳細情報（ネットワーク・マウント・環境変数など）を確認する。

```bash
docker inspect <container>
docker inspect <container> | jq '.[0].NetworkSettings.IPAddress'
```

リソース使用量をリアルタイムで確認する。

```bash
docker stats            # 全コンテナ
docker stats <container>
```

コンテナ内のプロセスを確認する。

```bash
docker top <container>
```

## ディスクを掃除する

```bash
docker system df        # 使用量の確認
docker builder prune    # ビルドキャッシュ削除
docker image prune -a   # 未使用イメージを全削除
docker system prune -a  # コンテナ・イメージ・キャッシュを一括削除
```
