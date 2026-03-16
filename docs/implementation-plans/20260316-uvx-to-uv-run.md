# uvx → uv run 移行: プロセスハング問題の修正

状態: 下書き
作成日: 2026-03-16
作成者: Claude

## 1. 目的

SKILL.md のコマンド実行を `uvx --from` から `uv run --directory` に切り替え、
`uvx` のクリーンアップ処理によるプロセスハングを解消する。

## 2. 背景

- Claude Code の Bash ツール経由で `uvx --from "${CLAUDE_SKILL_DIR}" agent-loop plan review ...` を実行すると、Python 側の処理が完了した後もプロセスが終了せず、Bash ツールの 5 分タイムアウトで強制終了される
- 計測により、LLM 呼び出し完了（~287s）→ ファイル書き出し→ stdout flush まで正常に完了しているが、その後 12 秒以上プロセスが終了しないことを確認
- `os._exit(0)` で Python を強制終了しても再現 → Python exit 処理は無罪
- `grep claude -p` で孤児プロセスなし → claude -p サブプロセスも無罪
- `uv run --directory` で同じコマンドを実行すると即座に終了 → **`uvx` の一時仮想環境クリーンアップがハングの原因**
- `uvx` は実行のたびに一時 venv を作成・破棄するのに対し、`uv run` はプロジェクトの `.venv` を再利用するため、終了処理が高速

## 3. 変更対象

- `skills/implementation-review-loop/SKILL.md`
- `README.md`
- `AGENTS.md`

## 4. 影響範囲

- SKILL.md を参照してコマンドを実行する全てのエージェント（Claude Code, Codex, Gemini）
- `allowed-tools` のパターンマッチが `Bash(uvx:*)` から `Bash(uv:*)` に変わる
- README.md / AGENTS.md のドキュメント例

## 5. 非対象範囲

- Python ソースコード（`src/agent_loop/`）の変更は行わない
- `pyproject.toml` の `[project.scripts]` エントリーポイントは変更しない
- `uv` 自体のバージョン要件は追加しない（既に前提条件として `uv` インストール済みを想定）
- 計測用 TIMING ログの追加・削除は本計画の範囲外

## 6. 実装方針

- `uvx --from "${CLAUDE_SKILL_DIR}"` を `uv run --directory "${CLAUDE_SKILL_DIR}"` に機械的に置換する
- `allowed-tools` のパターンを `Bash(uv:*)` に更新する
- `uvx` は `uv tool run` のエイリアスであり一時環境を使う。`uv run --directory` はプロジェクトディレクトリの `.venv` を使い、無ければ自動作成する。コマンドライン引数の構造は同一（`agent-loop <subcommand> ...`）なので互換性の問題は生じない

## 7. 実装手順

1. `skills/implementation-review-loop/SKILL.md`:
   - frontmatter の `allowed-tools` を `'Bash(uvx:*)'` → `'Bash(uv:*)'` に変更
   - 本文中の全 `uvx --from "${CLAUDE_SKILL_DIR}"` を `uv run --directory "${CLAUDE_SKILL_DIR}"` に置換（計 8 箇所: Procedure 5 箇所 + Command Templates 3 箇所）
2. `README.md`:
   - `uvx --from skills/implementation-review-loop` を `uv run --directory skills/implementation-review-loop` に置換
3. `AGENTS.md`:
   - `uvx --from skills/implementation-review-loop` を `uv run --directory skills/implementation-review-loop` に置換

## 8. 必須確認項目

- `skills/implementation-review-loop/SKILL.md` — 全 `uvx` 参照が `uv run` に置換されていること
- `README.md` — 使用例が更新されていること
- `AGENTS.md` — 説明が更新されていること
- リポジトリ内に `uvx` への参照が残っていないこと（`grep -r uvx` で確認）

## 9. 必須 checks

- `uv run --directory skills/implementation-review-loop agent-loop --help` が正常に動作すること
- `grep -r 'uvx' skills/ README.md AGENTS.md` の結果が空であること

## 10. 受け入れ条件

- SKILL.md 内に `uvx` への参照が一切ないこと
- `uv run --directory` でコマンドが正常に実行・終了し、タイムアウトしないこと
- `allowed-tools` が `Bash(uv:*)` に設定されていること
- README.md / AGENTS.md のドキュメントが更新されていること

## 11. エスカレーション条件

- `uv run --directory` で予期しないエラーが発生した場合（依存解決の差異など）
- `Bash(uv:*)` パターンが意図しないコマンドまで許可してしまう場合（セキュリティ上の懸念）

## 12. 実装役向けメモ

- 置換は機械的。`uvx --from` → `uv run --directory` の 1:1 置換のみ行う
- コマンド引数（`agent-loop plan review ...` 等）は一切変更しない
- `allowed-tools` パターン `Bash(uv:*)` は `uv` で始まるコマンド全てにマッチする。`uvx` より広い範囲だが、`uv` コマンド自体が安全なので実用上問題ない
