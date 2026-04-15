# Docker 個人知識ベース

## 1. セットアップ（Colima）

macOS では Docker Desktop の代替として Colima を使う。

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
パスは Homebrew のインストール先によって異なる場合があるため、`brew --prefix` で確認すること。

---

## 2. Colima VM 操作

### 手動での起動・停止

```bash
colima start
colima stop
```

### 自動起動の設定（ホスト再起動時に自動で `colima start`）

```bash
brew services start colima  # 有効化
brew services stop colima   # 無効化
```

---

## 3. ディスク・キャッシュ管理

ディスク使用量の確認

```bash
docker system df
```

ビルドキャッシュの削除

```bash
docker builder prune
```

---

## 4. データボリューム

### データボリュームコンテナの作成

busybox を使った軽量なデータボリュームコンテナを作成する

```bash
docker run --name=bb-container -v bb-volume:/shared busybox
```

- `/shared` がマウントポイント
- `bb-volume` ボリュームは自動で作成される

別のコンテナからボリュームをマウントして使う

```bash
docker run --rm --volumes-from bb-container -it ubuntu /bin/bash
```

```bash
cd /shared && echo 'hello world' > test.dat
```

> `docker container prune` で busybox コンテナをうっかり削除しても、ボリューム自体は残るので作り直せる。

### バックアップ

```bash
docker run --rm --volumes-from bb-container -v $HOME/backup:/backup ubuntu \
  tar cvf /backup/container-bkup.tar -C / shared
```

### リストア

```bash
docker run --rm --volumes-from bb-container -v $HOME/backup:/backup ubuntu \
  tar xvf /backup/container-bkup.tar -C /
```
