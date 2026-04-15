# Git 個人知識ベース

## リポジトリを始める

### グローバル設定

```bash
git config --global user.name "xion6"
git config --global user.email "u.xion6@gmail.com"
git config --global core.editor "vim"
```

設定内容の確認

```bash
git config -l
```

### エイリアス設定

設定ファイルを直接編集する方法

```bash
git config --global --edit
```

コマンドで個別に追加する方法

```bash
git config --global alias.chb "checkout -b"
git config --global alias.o10 "log --oneline -n 10"
git config --global alias.cm "commit -m"
```

### GitHub 連携セットアップ

```bash
git init
git add .
git commit -m "Initial commit"
```

GitHub CLI で認証してリポジトリを作成・接続

```bash
gh auth login
gh repo create {リポジトリ名} --private --description "{説明}"
git remote add origin https://github.com/xion6/{リポジトリ名}.git
git branch -M main
git push -u origin main
```

## 日常のコミット

```bash
git commit -m "メッセージ"
```

### fixup — 過去のコミットを修正する

> [!info] なぜ fixup か
> `--amend` はプッシュ済みのコミットに使うと履歴の書き換えが必要になる。fixup は新しいコミットとして積んでおき、後で rebase で統合するため、共有リポジトリでも安全。

1. 修正したいコミットのハッシュを確認する

```bash
git log --oneline -n 10
```

2. 修正内容をステージングし、fixup コミットを作成する

```bash
git add <修正ファイル>
git commit --fixup <対象コミットのハッシュ>
```

3. `rebase --autosquash` で fixup コミットを統合する

```bash
git rebase --autosquash -i <対象コミットの1つ前のハッシュ>
```

`fixup!` プレフィックスが付いたコミットが自動的に対象コミットの直後に移動・統合される。

## ブランチを扱う

### 切り替え

```bash
git switch <branch_name>
```

### ローカルブランチの最新化

```bash
git switch <local_branch>
git fetch
git merge origin/<remote_branch_name>
```

### 競合の解決フロー

マージ後に競合したファイルを確認する（先頭が `UU` のファイルが競合対象）

```bash
git status -s
```

競合を解消したファイルをステージングする（`UU` → `M` に変わる）

```bash
git add <競合ファイル名>
```

マージを続行または中止する

```bash
git merge --continue
git merge --abort
```

## リモートを管理する

### 確認・変更

```bash
git remote -v
git remote add origin https://github.com/<ユーザー名>/<リポジトリ名>.git
git remote set-url origin https://github.com/<ユーザー名>/<リポジトリ名>.git
```

### 上流ブランチの設定

> [!info] どのパターンをいつ使うか
> チェックアウト時: リモートブランチと対になるローカルブランチを新規作成するとき。
> プッシュ時: ローカルで作ったブランチを初めてリモートに送るとき。
> 既存ブランチに追跡させる: リモート名が変わったとき、または fetch 後に手動で紐づけたいとき。

**チェックアウト時に設定する**

```bash
git checkout -b foo origin/foo
git pull  # 上流設定済みなら引数不要
```

**プッシュ時に設定する**

```bash
git push -u origin <branch_name>
git push  # 以降は引数不要
```

**既存ブランチに追跡させる**

```bash
git fetch --all
git branch -u origin/<remote_branch_name>

# 例: bar ブランチに origin/foo を追跡させる
git branch -u origin/foo bar
```

上流ブランチの確認

```bash
git branch -vv
```

## 履歴を見る

```bash
git log --oneline -n 5       # 直近5件
git log --pretty=oneline     # 全ハッシュ付きで1行表示
```

## 並行して作業する（worktree）

> [!info] いつ使うか
> 同じリポジトリで別ブランチの作業を同時に進めたいとき。ブランチを切り替えると現在の作業が中断されるが、worktree なら別ディレクトリで並行して進められる。stash の代替として使える。

現在の作業ツリーのトップに移動

```bash
cd `git rev-parse --show-toplevel`
```

`../tmptree` を作成して `fix_bug` ブランチをチェックアウト

```bash
git worktree add ../tmptree fix_bug
```

作業ツリーの一覧確認

```bash
git worktree list
```

### 削除

注意: `.git` ディレクトリを持つ元の作業ツリーは削除しないこと。

```bash
rm -fr <tmptree までのパス>
```

削除対象の確認（ドライラン）→ 実行

```bash
git worktree prune --dry-run
git worktree prune --verbose
```
