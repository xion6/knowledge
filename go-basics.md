# Goプロジェクト 基本

Go初心者向け 最小手順。

## 1. Goインストール確認

```bash
go version
```

通ればOK。未導入なら公式サイトからインストール。

https://go.dev/dl/

## 2. プロジェクト作成

```bash
mkdir myapp
cd myapp
go mod init myapp
```

## 3. `main.go` 作成

```go
package main

import "fmt"

func main() {
	fmt.Println("hello")
}
```

## 4. 実行

```bash
go run .
```

## 5. ビルド

```bash
go build .
./myapp
```

## 最小構成

```text
myapp/
  go.mod
  main.go
```

## まず覚える項目

1. `package main` と `func main()`
2. `go mod init` で依存管理開始
3. `go run .` で実行
4. `go build .` でバイナリ作成
5. `go test ./...` でテスト実行

## よく使うコマンド

```bash
go fmt ./...
go test ./...
go mod tidy
```

## 最初に意識すること

- まず `main.go` 1個で始める
- 動いた後で分割
- 先に構成を凝らしすぎない
- `go fmt` を習慣化