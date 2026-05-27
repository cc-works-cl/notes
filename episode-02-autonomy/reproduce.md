---
title: 読者再現手順 — 自分で dummy リポジトリを clone して 3 ツール検証する
---

# 読者再現手順

本記事の数値を **自分の手で再現** したい方向けの手順です。dummy リポジトリ [`cc-works-cl/task-mesh-tag-mvp`](https://github.com/cc-works-cl/task-mesh-tag-mvp) は Public で、MIT ライセンスです。

## 前提

- macOS / Linux (Stop.sh は bash 想定、Windows は WSL2 推奨)
- Node.js 20 + pnpm 9
- Docker (Postgres 16 + pg_trgm 用)
- 以下のいずれかのツール契約 (3 つそろえる必要はなし、1 つから始められます)
  - Claude Code (Claude Max 5x = 月 $100 から)
  - Codex (ChatGPT Plus = 月 $20 から)
  - Antigravity (Google AI Pro 連携)

## ステップ 1: 環境を立ち上げる

```bash
# clone
git clone https://github.com/cc-works-cl/task-mesh-tag-mvp.git
cd task-mesh-tag-mvp

# Postgres 起動 (pg_trgm 拡張入り)
docker compose up -d postgres

# 依存と migration
pnpm install
cp .env.example .env
pnpm prisma migrate dev

# 初期テストが緑であることを確認
pnpm test
```

`health check` が 1 件 pass で緑なら準備完了です。

## ステップ 2: AI ツールに Issue を投げる

リポジトリの GitHub Issues から、放置 OK タスク 4 本 (#1-#4) のいずれかを選んで AI ツールに投入します。

### Claude Code (CLI)

```bash
# tmux でスリープしても継続するセッション
caffeinate -i tmux new -s overnight

# tmux 内で
cd task-mesh-tag-mvp
claude

# Issue #1 の本文をコピペして投入
# (Issue: https://github.com/cc-works-cl/task-mesh-tag-mvp/issues/1)
# Ctrl-B → D で detach、寝室へ
```

### Codex (ブラウザ)

1. ChatGPT で Codex ダッシュボードを開く
2. GitHub 連携で `cc-works-cl/task-mesh-tag-mvp` を許可
3. Issue #4 の本文をコピペして「新規タスク」として投入
4. 2 タスク並列にしたい場合は Issue #2 も別タスクで投入

### Antigravity (IDE)

1. Antigravity を起動して `task-mesh-tag-mvp` を開く
2. Issue #3 の本文を Cloud 面に投入
3. Browser Subagent で Storybook の動作確認まで自走させる

## ステップ 3: 朝に数値を取得

```bash
# tmux 再 attach (Claude Code の場合)
tmux attach -t overnight

# Claude Code を Ctrl-C で終了

# 数値を JSONL に
./scripts/morning-collect.sh claude-code "2026-05-28 22:00:00" >> my-run.jsonl
```

出力例:
```json
{"ts":"...","kind":"coding-agent-overnight","tool":"claude-code","commits":<N>,"self_run_minutes":<N>,"tests_generated_pass":<N>}
```

## ステップ 4: 朝確認 5 項目チェックリスト

note 本編の h2-6 で書いた 5 項目を上から順に通します:

1. 依存ライブラリの追加 (`package.json` 差分): `git diff main..<branch> -- package.json`
2. データベース変更 (down が用意されているか): `prisma/migrations/` の差分
3. テストの合格率: `pnpm test`
4. 設計判断の妥当性 (責務分割 / 命名 / 層またぎ): `git diff` で目視
5. コミットの粒度: `git log --oneline`

合計 30 分で 1 PR の提出可否を判定します。

## 失敗例の再現

failure-replay ラベルの Issue (#F1 / #F2 / #F3) は意図的に失敗を踏ませるトラップです。再現手順は以下を参照:

- [F1: 無限ループ (pg_trgm)](./failure-replay/F1-infinite-loop-pg-trgm)
- [F2: 設計判断ミス (Tag × Role)](./failure-replay/F2-design-judgment-tag-role)
- [F3: テスト書けない (LLM 出力)](./failure-replay/F3-llm-test-not-writable)

---

[← episode-02-autonomy ハブに戻る](./)
