# Goプロジェクト 発展

Goプロジェクトのフォルダ構成を、少し大きい開発向けに整理。

## 少し慣れた後の構成

```text
myapp/
  cmd/
    myapp/
      main.go
  internal/
  pkg/
  go.mod
```

使い分け 目安:

- 小さい学習用 → `main.go` 1個
- 作り始め 本番寄り → `cmd/myapp/main.go`
- 外部公開しない共通処理 → `internal/`
- 外部公開して再利用したいライブラリ → `pkg/`

## それぞれ 何を置くか

### `cmd/`

アプリの起動入口を置く場所。実行ファイルごとにディレクトリを分ける。

例:

```text
cmd/
  myapp/
    main.go
  batch/
    main.go
```

- `myapp` → Webサーバー起動
- `batch` → 定期実行バッチ起動

つまり `cmd/` 配下は「何を起動するか」を並べる場所。

### `internal/`

プロジェクト内部だけで使うコードを置く場所。他プロジェクトから import されにくくするための仕組み。

例:

```text
internal/
  user/
    service.go
    repository.go
  db/
    mysql.go
```

- `internal/user/service.go` → ユーザー処理
- `internal/user/repository.go` → DB読み書き
- `internal/db/mysql.go` → DB接続

アプリの本体ロジックは、まず `internal/` に入れる考え方で十分。

### `pkg/`

他プロジェクトからも再利用される前提のコードを置く場所。

例:

```text
pkg/
  retry/
    retry.go
```

ただし初心者段階では、`pkg/` は無理に使わなくていい。外部公開前提が明確なときだけ使う。

## 典型的 分け方

### 学習用・小さいツール

```text
myapp/
  go.mod
  main.go
```

最初はこれで十分。関数が増えたら同じディレクトリに `foo.go` を追加してもいい。

### 少し大きいアプリ

```text
myapp/
  cmd/
    myapp/
      main.go
  internal/
    handler/
      user.go
    service/
      user.go
    repository/
      user.go
    db/
      mysql.go
  go.mod
```

流れ:

- `cmd/myapp/main.go` → 起動
- `internal/handler/` → HTTP受け取り
- `internal/service/` → 業務ロジック
- `internal/repository/` → DBアクセス
- `internal/db/` → DB接続設定

Web APIなら この分け方でかなり進められる。

## 分割 タイミング

- 最初から `cmd/` `internal/` `pkg/` 全部作らなくていい
- `main.go` が長くなったら分割
- 他のファイルからも使う処理が出たら `internal/` へ移動
- 起動方法が複数になったら `cmd/` を作る
- 外部公開したい共通部品だけ `pkg/` に出す

## 初心者向け おすすめ

最初は 次の順で十分。

1. `main.go` 1個
2. ファイル数 増えたら 同じ階層に分割
3. アプリらしくなったら `cmd/` と `internal/` 導入
4. `pkg/` は必要になるまで作らない