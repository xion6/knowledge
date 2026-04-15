# Git 個人知識ベース

## 1. 初期設定

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

リポジトリ初期化

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

---

## 2. リモート操作

### リモートリポジトリの確認

```bash
git remote -v
```

### リモートリポジトリの登録

```bash
git remote add origin https://github.com/<ユーザー名>/<リポジトリ名>.git
```

### リモートリポジトリのアドレス変更

```bash
git remote set-url origin https://github.com/<ユーザー名>/<リポジトリ名>.git
```

### 上流ブランチの設定

**チェックアウト時に設定する**

ローカルブランチを作成し、`origin/foo` を上流ブランチとして設定

```bash
git checkout -b foo origin/foo
```

上流ブランチが設定済みの場合は省略できる

```bash
git pull origin foo  # 未設定の場合
git pull             # 設定済みの場合
```

**プッシュ時に設定する**

`-u` / `--set-upstream` オプションで上流ブランチを設定

```bash
git push -u origin <branch_name>
git push  # 設定済みの場合
```

**既存のローカルブランチにリモートブランチを追跡させる**

```bash
git fetch --all
git branch -u origin/<remote_branch_name>

# 例: bar ブランチに origin/foo を追跡させる
git branch -u origin/foo bar
```

### 上流ブランチの確認

```bash
git branch -vv
```

---

## 3. ブランチ操作

### ブランチの切り替え

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

競合を解消したファイルをステージングする（`UU` → `M` マークに変わる）

```bash
git add <競合ファイル名>
```

マージを続行する

```bash
git merge --continue
```

マージ作業を中止する

```bash
git merge --abort
```

---

## 4. コミット操作

### 通常コミット

```bash
git commit -m "メッセージ"
```

### --fixup: 過去のコミットを修正する

指定したコミットに対する修正コミットを作成し、後で `rebase --autosquash` で自動統合できる。

**手順**

1. 修正したいコミットのハッシュを確認する

```bash
git log --oneline -n 10
```

2. 修正内容をステージングし、fixup コミットを作成する

```bash
git add <修正ファイル>
git commit --fixup <対象コミットのハッシュ>
```

3. `rebase --autosquash` で fixup コミットを対象コミットに統合する

```bash
git rebase --autosquash -i <対象コミットの1つ前のハッシュ>
```

> `fixup!` プレフィックスが付いたコミットが自動的に対象コミットの直後に移動・統合される。

---

## 5. ログ・履歴確認

直近 5 件をコンパクトに表示

```bash
git log --oneline -n 5
```

全ハッシュ付きで 1 行表示

```bash
git log --pretty=oneline
```

---

## 6. 作業ツリー（git worktree）

現在の作業ツリーのトップディレクトリに移動

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

> この作業ツリーでは通常と同様にコミット・プッシュ・プルが実施できる。

### 作業ツリーの削除

注意: `.git` ディレクトリを持つ元の作業ツリーは削除しないこと。

```bash
rm -fr <tmptree までのパス>
```

削除対象の確認（ドライラン）

```bash
git worktree prune --dry-run
```

実際に prune を実行

```bash
git worktree prune --verbose
```
