# headroom 個人知識ベース

LLM へ送る前にツール出力・ログ・検索結果を圧縮してトークンを削る最適化レイヤ。CLI ＋ ライブラリ ＋ MCP サーバー ＋ プロキシとして動く。PyPI 名は `headroom-ai`。

## インストールする（uv tool で隔離）

```bash
uv tool install --python 3.13 "headroom-ai[proxy]"
headroom --version
```

> [!info] base だけでは動かない
> base 版は CLI 起動時に proxy モジュール（fastapi 依存）を無条件 import するため、`[proxy]` extra が無いと `--help` すら失敗する。最小構成でも `[proxy]` を付ける。

> [!info] どの使い方を選ぶか
> - API キー（`ANTHROPIC_API_KEY` 等）で動かしているなら「ラップして一発で使う」が最短。
> - 任意のクライアントに繋ぎたい・細かく制御したいなら「プロキシを立てて手動で繋ぐ」。
> - API アクセスが無いサブスク（Pro/Max）運用なら「MCP で必要なときだけ原文を取り戻す」。

## ラップして一発で使う（最短）

プロキシ起動・環境変数設定・ツール起動をまとめて代行する。

```bash
headroom wrap claude     # Claude Code を Headroom 経由で起動
```

対応: `claude` `codex` `aider` `copilot` `goose` `cursor` `cline` `continue` `openhands`。
※ API キー方式（`ANTHROPIC_API_KEY` 等）が前提。サブスク運用は MCP 方式（後述）へ。

## プロキシを立てて手動で繋ぐ

別ターミナルでプロキシを立て、クライアントの向き先を変えるだけ。

```bash
headroom proxy                                     # port 8787 で起動
ANTHROPIC_BASE_URL=http://localhost:8787 claude    # Anthropic 系
OPENAI_BASE_URL=http://localhost:8787/v1 your-app  # OpenAI 互換
```

主なオプション:
- `--port 8080` ポート変更
- `--mode token`（既定・圧縮優先）/ `--mode cache`（プレフィックスキャッシュ命中率優先）
- `--no-optimize` 素通し（デバッグ用）

## MCP で必要なときだけ原文を取り戻す（CCR）

要約だけ先に渡し、詳細が要るときだけ `headroom_retrieve` で原文を取り戻す。API アクセスが無いサブスク利用者でも使える。

```bash
headroom mcp install --agent claude   # Claude Code に MCP サーバー登録
headroom proxy                        # 別ターミナルで圧縮バックエンド起動
headroom mcp status                   # 状態確認
```

確認は `headroom mcp status` より `claude mcp list` / `claude mcp get headroom`（"✓ Connected" が出れば OK）。登録後は Claude Code を再起動して反映する。全トラフィックの自動圧縮まで効かせるならプロキシ方式の `ANTHROPIC_BASE_URL` も併用。

> [!warning] status の Claude Config ✗ は誤検知
> `headroom mcp status` は旧形式 `~/.claude/mcp.json` だけを見る。Claude Code 2.x は `~/.claude.json` に保存するため ✗ と出るが、`claude mcp get headroom` が Connected なら実害なし。

## コマンドを早引きする

```bash
headroom wrap <tool>          # ツールをプロキシ経由で起動
headroom proxy                # プロキシ単体を起動
headroom mcp install/status/uninstall   # MCP 登録の管理
headroom learn [--apply]      # 過去の失敗パターンを学習し再発防止コンテキスト生成
headroom memory list/show/stats/export  # 蓄積メモリの管理
headroom perf                 # ログからプロキシ性能・削減量を分析
headroom init claude -g       # フック/ルーティングを恒久インストール
```

## アップデート・削除する（uv tool）

```bash
uv tool upgrade headroom-ai     # 更新
uv tool uninstall headroom-ai   # 削除（隔離環境ごと消える）
```
