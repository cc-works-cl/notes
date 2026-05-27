---
title: Claude Code 一晩自走の生ログ — 第2話【自走実装編】
---

# Claude Code 一晩自走の生ログ

本ページは note 本編 (第2話) の line 110 で参照される **Claude Code 単独実走の raw log** を置く場所です。

## 検証環境

- リポジトリ: [`cc-works-cl/task-mesh-tag-mvp`](https://github.com/cc-works-cl/task-mesh-tag-mvp)
- ホスト: macOS (Apple Silicon)
- Node.js 20 / pnpm 9 / Postgres 16 (Docker)
- Claude Code: Claude Max 20x プラン、モデルは Claude Opus
- 投入 Issue: #1 (Tag CRUD 4 API) + #2 (タグ件数集計エンドポイント)

## 実走スケジュール (予定)

| 日時 | フェーズ | 内容 |
|---|---|---|
| 2026-05-28 (本日昼) | A-pre | Stop.sh フックの単体動作確認済み (exit 0 / exit 1 + STOP メッセージ) |
| 2026-05-28 22:00 | B-1 投入 | `tmux new -s overnight-cc` → `caffeinate -i claude` → 30 行仕様 + 運用指示貼付 |
| 2026-05-29 07:00 | B-1 朝確認 | `./scripts/morning-collect.sh claude-code` で数値取得 |
| 2026-05-29 朝以降 | D-1 反映 | 本文 line 110 の数値を実測値で差し替え |

## 取得予定の数値

`./scripts/morning-collect.sh claude-code "2026-05-28 22:00:00"` で以下が JSONL に追記される:

```json
{
  "ts": "<ISO8601>",
  "kind": "coding-agent-overnight",
  "sprint_id": "2026-07-W27-ai-coding-mapping-autonomy",
  "tool": "claude-code",
  "repo": "cc-works-cl/task-mesh-tag-mvp",
  "issue_ids": ["#1", "#2"],
  "prep_minutes": <投入準備 (分)>,
  "self_run_minutes": <自走時間 (分)>,
  "wall_clock_minutes": <壁時計 (分)>,
  "commits": <コミット数>,
  "tests_generated_total": <生成テスト本数>,
  "tests_generated_pass": <pass 数>,
  "tests_existing_reg": <既存テストの壊れ数>,
  "morning_fix_minutes": <朝修正 (分)>,
  "failure_mode": null,
  "verdict": "pass"
}
```

## 実測値 (取得後に追記)

> 2026-05-29 朝、実走完了時に本セクションを更新します。

```jsonl
(placeholder: B-1 実走完了後に JSONL 1 行を貼り付け)
```

### コミット履歴の抜粋

```bash
$ git log --oneline --since="2026-05-28 22:00" main..feat/tags-crud
(placeholder: 朝の git log 出力を貼り付け)
```

### vitest 出力の抜粋

```bash
$ pnpm test --reporter=verbose
(placeholder: vitest 出力を貼り付け)
```

### package.json diff (依存追加の確認)

```bash
$ git diff main feat/tags-crud -- package.json
(placeholder: 依存追加があれば貼り付け)
```

## 観察コメント

(placeholder: 実測完了後に「期待値とのギャップ」「Stop.sh が発火したか」「設計判断ミスの有無」を 200-400 字でまとめる)

---

[← episode-02-autonomy ハブに戻る](./)
