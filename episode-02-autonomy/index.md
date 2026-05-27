---
title: 第2話【自走実装編】 補足ノート — AIコーディング使い分けマップ
---

# 第2話【自走実装編】 補足ノート

note 本編「副業エンジニアのためのAIコーディング使い分けマップ【自走実装編】一晩で1機能を仕上げる」の補足素材です。新シリーズ `ai-coding-mapping-2026` の第 2 話、Claude Code / Codex / Antigravity を一晩自走させて朝に PR の質を確認するワークフローの実走ログと再現手順を置きます。

## 検証用 dummy リポジトリ

本記事の数値は、検証用に切り出した dummy リポジトリで取った実測です:

- **GitHub**: [`cc-works-cl/task-mesh-tag-mvp`](https://github.com/cc-works-cl/task-mesh-tag-mvp) (Public, MIT)
- スタック: React 19 + Vite + Node.js 20 + Express + TypeScript + Prisma + Postgres 16 (pg_trgm) + vitest + Storybook 8
- Issue 11 本 (放置 OK 4 / 放置 NG 4 / failure-replay 3)
- フィクション宣言: 受託案件の実コードではなく、検証目的の dummy です ([DISCLAIMER.md](https://github.com/cc-works-cl/task-mesh-tag-mvp/blob/main/DISCLAIMER.md))

## 収録コンテンツ

- [Claude Code 一晩自走の生ログ](./verification-log) — Issue #1 + #2 を投入して朝に取った数値の raw log。本記事 line 110 の数値の出所
- [失敗例 F1 (無限ループ) の再現](./failure-replay/F1-infinite-loop-pg-trgm) — `tag_search.ts` への 5 連続コミットと Stop.sh フックの停止挙動
- [失敗例 F2 (設計判断ミス) の再現](./failure-replay/F2-design-judgment-tag-role) — Tag リポジトリへの認可ロジック染み出しを意図的に起こす Issue 設計
- [失敗例 F3 (テストが書けない) の再現](./failure-replay/F3-llm-test-not-writable) — LLM 出力のテストモック方針不在で Claude が判断保留する挙動
- [読者再現手順](./reproduce) — dummy リポジトリを clone して自分で 3 ツール検証を回す手順

## Codex / Antigravity の検証状況

本記事公開時点 (2026-05-28) では Claude Code 単独の実走数値のみ本文に書いています。Codex / Antigravity の dummy リポジトリ実走は週単位で進行中で、結果が出次第 [verification-log](./verification-log) に追記し、本文も note 編集機能で更新します。

## 本編記事

note で公開しています: 副業エンジニアのためのAIコーディング使い分けマップ【自走実装編】(URL は公開と同時に追記)

- 第1話【ツール俯瞰編】: [note.com/cc_works_cl/n/ne2f09b4fd2dd](https://note.com/cc_works_cl/n/ne2f09b4fd2dd)
- 第3話【協働編】(予告): 執筆中
- 第4話【テンプレ集 50 選】(有料 1,280 円予定、執筆中)

---

[← シリーズ全体のインデックスに戻る](../)
