---
title: Claude Code 一晩自走の生ログ — 第2話【自走実装編】
---

# Claude Code 一晩自走の生ログ

note 本編 (第2話) の line 110-118 で参照される **Claude Code 単独実走の raw log** です。

## 検証環境

- リポジトリ: [`cc-works-cl/task-mesh-tag-mvp`](https://github.com/cc-works-cl/task-mesh-tag-mvp)
- PR: [#12 feat(tags): Tag CRUD 4 API + GET /tags/stats](https://github.com/cc-works-cl/task-mesh-tag-mvp/pull/12)
- ブランチ: `feat/tags-crud-and-stats`
- ホスト: macOS (Apple Silicon)
- Node.js v22.14.0 / pnpm 11.4.0 / Postgres 16 (Colima + docker-compose V1)
- Claude Code: Claude Max 20x プラン、モデル Claude Opus 4.7 (1M context)
- 投入 Issue: [#1 Tag CRUD 4 API](https://github.com/cc-works-cl/task-mesh-tag-mvp/issues/1) + [#2 タグ件数集計エンドポイント](https://github.com/cc-works-cl/task-mesh-tag-mvp/issues/2)

## 実走サマリ

| 指標 | 実測値 |
|---|---|
| 投入準備 + プラン承認 | 約 2 分 |
| **コミット時刻ベース自走時間** | **3 分 24 秒** (00:55:21 → 00:58:45 JST) |
| Cooked (チャット セッション total) | 9 分 26 秒 |
| 壁時計 (人間が見るまでの待ち) | 約 6 時間 (00:58 完了 → 朝 7 時頃確認) |
| コミット数 | **10** |
| 追加行 / 削除行 | +454 / -9 |
| ファイル数 | 8 |
| 生成テスト | 15 本 (tags 12 + tags-stats 3) |
| 既存テスト reg | **0 件** (health.test.ts 1 本緑維持) |
| テスト pass 率 | **16/16 (100%)** |
| 朝修正時間 | **0 分** (16/16 緑のため修正不要) |
| 依存追加 | 0 件 (allowlist 厳守) |
| Stop.sh 発火 | なし (同一ファイル連続コミット最大 2、5 未満) |

## コミット履歴 (1 関心事 / 1 コミット)

```bash
$ git log --oneline --reverse main..feat/tags-crud-and-stats
4e1c47e feat(repository): add tag-repository with CRUD methods
024b0c3 feat(service): add tag-service with zod validation
b45e611 feat(api): add tags router with POST/GET/PATCH/DELETE
6898508 feat(app): mount tags router in createApp
9036a2b test(tags): cover CRUD 4 endpoints with supertest
5525183 feat(repository): add count and groupBy methods to tag-repository
f084542 feat(api): add tags-stats router with GET /stats
3216977 feat(app): mount tags-stats router before tags router
13a2de0 chore(test): disable vitest file parallelism
b566a6c test(tags-stats): seed 3 users and assert aggregate counts
```

Repository → Service → API → mount → test → (集計版) Repository → API → mount → test config → test の **層別ボトムアップ順** で進んでいる。`git revert <SHA>` で単位ごとに巻き戻せる粒度。

## 設計判断 (Claude Opus が判断保留で出してきた 3 件)

CLAUDE.md の責務 3 行ルール + 判断保留ルールが効いて、Claude Opus は plan 段階で以下 3 つを「人間に承認を求めます」と提示してきた:

1. **ownerId は `X-Owner-Id` ヘッダ伝達**
   - 理由: Issue #1 の POST body スキーマ `{ name, description?, scope? }` を変更しない要件と、認可ロジックを Service / API 層に分離したい要件の整合を取るため
   - 承認後: `b45e611 feat(api): add tags router with POST/GET/PATCH/DELETE`

2. **`/tags/stats` を `/tags/:id` より先にマウント**
   - 理由: Express のルートマッチング順序で `/tags/:id` が `id="stats"` を吸ってしまうとルート競合
   - 承認後: `3216977 feat(app): mount tags-stats router before tags router`

3. **集計は `Prisma.groupBy(by: ['ownerId'])` で N+1 回避**
   - 理由: ownerId ごとに count を `findMany` でループ取得すると N+1、Prisma の `groupBy` で 1 クエリに集約
   - 承認後: `5525183 feat(repository): add count and groupBy methods to tag-repository`

これらが「**判断保留ルール (`docs/CLAUDE.md`)** が機能した実例」で、AI が独断で設計判断をすり抜けるのを構造的に防いだ成功パターン。

## 計画外追加 (vitest.config.ts、ユーザ承認済)

複数 PrismaClient (test ファイル並列実行で生成) による Postgres 接続競合で **3 件のテストが落ちた** → AI が「`fileParallelism: false` を設定すれば解消できる、これは依存追加ではなく既存 vitest の設定ファイル」と plan で提案、ユーザ承認後にコミット (`13a2de0`):

```typescript
// vitest.config.ts (新規追加)
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    fileParallelism: false,
  },
});
```

CLAUDE.md の依存 allowlist は変更なし、vitest 自体は既に devDependencies に入っているため設定ファイルの追加のみで完結。

## 観察コメント

- **Claude Opus 4.7 の速さ**: 3 分 24 秒で 10 コミット = 1 コミットあたり 20 秒。ファイル間の依存関係を理解した上での順次実装で、待ち時間がほぼ無し
- **「夜実装する → 朝確認する」物語との整合**: AI 自走は 3 分 24 秒で完了、その後ブランチが緑のまま **約 6 時間静止** して朝のレビューを待つ → 「壁時計で夜から朝」の物語構造は維持
- **責務 3 行ルール厳守**: `tag-repository.ts` に `RoleService` や `User` の import が一切なく、Tag のみ扱う Repository の純粋性が保たれている
- **コミット粒度**: 同一ファイル連続コミットは `src/index.ts` の 2 回 (tags router mount + stats router mount) のみで、Stop.sh フック (5 連続で停止) は不発火。CLAUDE.md ルール通り

## reproduce 手順

このログの数値を読者の方が自分で再現する手順は [reproduce](./reproduce) を参照。基本的には `git clone` → `docker-compose up -d postgres` → `pnpm install` → `pnpm prisma migrate dev` → Claude Code に Issue #1 + #2 を投入、で同じ結果が得られる想定です (Opus 4.7 のスピードは AI 側のバージョンによって変動する可能性あり)。

---

[← episode-02-autonomy ハブに戻る](./)
