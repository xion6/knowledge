# Python 個人知識ベース

macOS 上の Python は mise で管理し、グローバル環境は汚さない方針。CLI ツールは uv で隔離して入れ、グローバルは `pip` と `uv` だけの状態を保つ。

## 現在の Python 環境を把握する

何を消す・どこに入れるかを判断する前に、必ず現状を確認する。

```bash
which python python3 pip      # 実体のパスを確認
echo "$VIRTUAL_ENV"           # venv が有効か（空なら無効）
pip --version                 # どの Python に紐づく pip か
```

> [!info] mise 管理の Python とは
> グローバルの `python` は mise が管理する `~/.local/share/mise/installs/python/<version>/` 配下を指す。OS の Python ではなくユーザー環境なので、パッケージを消しても OS は壊れない。ただし venv ではないため、ここに直接入れたものは全プロジェクトで共有される。

## グローバルの pip パッケージを掃除する

venv を使わずグローバルに入れてしまったパッケージを一掃したいとき。

### まずバックアップ

```bash
pip freeze > ~/pip-packages-backup-$(date +%Y%m%d).txt
```

復元は `pip install -r <backup>` でできる。

### pip / uv を残して全削除

```bash
pip list --format=freeze \
  | grep -ivE '^(pip|uv)==' \
  | cut -d= -f1 \
  | xargs -r pip uninstall -y
```

> [!warning] pip freeze と pip 本体
> `pip freeze` は pip 自身を出力しないが、`pip list --format=freeze` は全部出す。後者を使い、残したいものを `grep -v` で除外するのが確実。

## CLI ツールを隔離環境に導入する（uv tool）

PyPI 上の CLI ツールは、グローバルに `pip install` せず `uv tool install` で入れる。ツールごとに専用 venv が作られ、依存が他と衝突しない（pipx 相当）。

```bash
uv tool install <package>                          # 基本
uv tool install --python 3.13 <package>            # Python を固定
uv tool install --python 3.13 "<package>[extra]"   # extras 付き
```

実体は `~/.local/share/uv/tools/<package>/`、実行ファイルは `~/.local/bin/` に置かれる（PATH 済みなら即使える）。

> [!tip] 新しすぎる Python に注意
> Rust 拡張や ML 依存を含むパッケージは、リリース直後の Python（例: 3.14）向け wheel が無いことが多く、その場合ソースビルドで失敗する。安定版（3.12 / 3.13）を `--python` で固定すると回避できる。対応版は PyPI の `requires-python` と classifiers で確認する。

### 例: headroom を入れる

```bash
uv tool install --python 3.13 "headroom-ai[proxy]"
```

> [!info] base だけでは動かないことがある
> headroom は base 版でも CLI 起動時に proxy モジュール（fastapi 依存）を無条件 import するため、`fastapi` が無いと `--help` すら失敗する。最小限 `[proxy]` extra を足すと解消した。「動かない＝足りない extra を探す」典型例。

### 導入後の確認

```bash
which <command>          # ~/.local/bin に入ったか
<command> --version      # 起動するか
```

## uv tool を管理する

```bash
uv tool list                 # 入れたツール一覧
uv tool upgrade <package>    # 個別アップグレード
uv tool upgrade --all        # 全ツール更新
uv tool uninstall <package>  # 削除（隔離 venv ごと消える）
```

## 隔離方式の使い分け

| 用途 | 方法 |
| --- | --- |
| CLI ツールとして常用する | `uv tool install`（専用 venv ＋ PATH に実行ファイル） |
| コードから import して使う | プロジェクトごとに venv を作る（`uv venv` / `python -m venv`） |
| 一度きり試すだけ | `uvx <package>`（インストールせず実行） |

> [!info] グローバルに入れない理由
> グローバル（mise）に `pip install` すると依存が混ざり、後で「どれが何のためか」が分からなくなる。CLI は uv tool で隔離、ライブラリは venv、使い捨ては uvx、と分けると、グローバルを `pip` と `uv` だけの綺麗な状態に保てる。
