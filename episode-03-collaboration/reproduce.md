---
title: 読者再現手順 — 手元のリポで Claude Code 4 工程協業を試す
---

# 読者再現手順 — `tag_color_preference` 相当のタスクを 4 工程で着地させる

note 本編 (第3話) と [verification-log](./verification-log) の数値を、手元の Node + Express + Postgres リポジトリで再現するための手順です。本記事の題材は架空タスクメッシュ案件の `User.tag_color_preference` カラム追加 1 タスクですが、構造が同じなら他スタック (Rails / Django / Phoenix など) でも置き換えて回せます。

## 前提

- macOS / Linux (Windows は WSL2 推奨)
- Node.js 20 + pnpm 9 (npm / yarn でも可)
- Docker (Postgres 16 用)
- Claude Code (Anthropic、Max プラン推奨だが Pro 単月でも可)
- 手元に既存の Node + Express + Knex + Postgres のリポジトリが 1 本ある状態 (個人開発の MVP でも、副業案件で手を入れている dummy リポでも可)

すでに自走実装編 (第2話) で使った `cc-works-cl/task-mesh-tag-mvp` をお持ちなら、そのまま流用できます。

```bash
git clone https://github.com/cc-works-cl/task-mesh-tag-mvp.git
cd task-mesh-tag-mvp
docker-compose up -d postgres
pnpm install
cp .env.example .env
pnpm prisma migrate dev
pnpm test  # health check が緑なら準備完了
```

## ステップ 1: ノートに 8 行を書く (10 分)

紙のノートでも Markdown ファイルでも構いません。4 点 (インターフェース / データモデル / 依存追加 / テスト粒度) を 8 行以内に圧縮します。本サイトの題材だと、次のような 8 行になります。

```
- API: PATCH /users/me { tag_color_preference: string (hex 7 文字) }
- DB: users テーブルに tag_color_preference カラム追加 (default '#F4A45D', nullable false)
- migration: UP は ADD COLUMN、DOWN は DROP COLUMN を両方書く
- 依存追加: なし (既存の express + knex でまかなう)
- 認可: 認証済みユーザーが自分の行のみ更新できる (既存 middleware を流用)
- バリデーション: hex 7 文字の正規表現で弾く
- テスト: integration 1 本 (PATCH 200 / 422 / 401 の 3 ケース)
- 影響範囲: User モデル + migration 1 本 + handler 1 本のみ
```

別のタスク (`Article` モデルに `pinned_at` を足す、など) に置き換える場合は、上の枠組みをそのままコピペし、各行の中身だけ書き直してください。

## ステップ 2: Claude Code に計画させる (5 分)

`claude` で対話セッションを開き、上の 8 行をそのまま貼り付けたあと、次の 1 行を続けて打ちます。

```
この設計メモに沿って計画を 6 ステップ以内で。判断が割れる箇所は「人間に承認を求めます」で止めてください。
```

数十秒で 6 ステップが返ってきます。途中で 1〜2 件「人間判断」を求められたら、1 行ずつ即答してください。本サイトの観察では、6 ステップのうち 1 件 (「既存ハンドラを拡張するか新規ハンドラを作るか」) で止まる傾向があります。

## ステップ 3: AI 自走に渡す (人間 0 分、AI 1 時間 30 分〜2 時間)

承認した 6 ステップを Claude Code に「進めて」とだけ書いて投げます。SubAgent を 2 本追加する場合は、`/agent` などのコマンドで次のような構成を組みます。

- **テスト並行 SubAgent**: 「メインが実装している裏で、integration テストの 3 ケース (200 / 422 / 401) を supertest で先に書いておいて」
- **セルフレビュー SubAgent**: 「実装が終わったら、`認可と null チェック` の観点で diff を 1 回セルフレビューして」

ここで `tmux new -s overnight` 等で session を detach すれば、画面を消したまま走らせられます。`caffeinate -i` をかぶせると macOS のスリープも防げます。

## ステップ 4: 朝にセルフレビューを回す (15 分)

AI 自走が終わっていたら、tmux に re-attach して次のプロンプトでセルフレビューを依頼します ([verification-log](./verification-log) と同じ全文)。

```
今書いた diff を、以下 3 点の観点で 1 回セルフレビューしてください。
- 設計メモ (ノート 8 行) との整合 — 外れている箇所があれば指摘
- 副作用 — git diff --stat で想定外のファイルが触られていないか
- テスト — 既存テストへの reg、生成テストの assertion の弱さ
判断が割れる箇所は「未決」ラベルで挙げて、私の判断を待ってください。
```

返ってくる未決事項は 2〜4 件、そのうち 1〜2 件で人間判断が要ります。残りは「既存規約に合わせて」と 1 行返せば Claude Code 側で吸収します。

## ステップ 5: PR 下書きを書かせて 3 行だけ直す (5 分)

`git push` の前に PR 下書きを生成させます。

```
このブランチの git log と diff から PR 説明文を 24 行以内で書いてください。
構造は タイトル / 背景 / 変更点 / 動作確認 / レビュー観点 / 関連 Issue。
背景と動作確認とレビュー観点は箇条書きで、私が後から 1 行ずつ書き直す前提。
タイトルは Conventional Commits (feat: ◯◯ のように種別 prefix を付ける書式) で。
```

返ってきた 24 行のうち、人間が直すのは次の 3 行だけです。

- **背景**: なぜ今このカラムが必要かの業務側の文脈 (AI には書けない 1 行)
- **動作確認**: 自分の手元で確認した手順を、レビュアーが再現できる形に
- **レビュー観点**: レビュアーに見てほしい 1 点を指名する

[verification-log](./verification-log) の「人間が直した 3 行 (before → after)」に diff を載せてあります。

## ステップ 6: 当日確認 5 項目チェック (12 分前後)

PR を出す前に、note 本編 h2-7 の 5 項目を上から順に通します。本サイトでは [verification-log](./verification-log) と本編で同じ順序にしてあります。

1. **設計整合** — ノートに書いた 8 行と diff が一致しているか
2. **副作用範囲** — `git diff --stat` で想定外のファイルが触られていないか
3. **テスト pass 率** — `pnpm test` を 1 回回し、既存テスト reg なし + 生成テストが緑
4. **PR 説明文の 3 行** — 背景・動作確認・レビュー観点が自分の言葉で書き直されているか
5. **コミット粒度** — 1 コミット = 1 関心事になっており、`git revert <SHA>` で単位ごとに巻き戻せるか

合計 12 分前後で PR の Approve まで届きます。

## 雛形 — DDL とハンドラ最小構成

手元に既存リポがなく、空のディレクトリから始めたい場合の最小構成を置いておきます。Knex + Express の組み合わせで、本サイトの 4 工程をそのまま回せます。

### マイグレーション (db/migrations/20260528_add_tag_color_preference.ts)

```ts
import type { Knex } from "knex";

export async function up(knex: Knex): Promise<void> {
  await knex.schema.alterTable("users", (t) => {
    t.string("tag_color_preference", 7)
      .notNullable()
      .defaultTo("#F4A45D");
  });
}

export async function down(knex: Knex): Promise<void> {
  await knex.schema.alterTable("users", (t) => {
    t.dropColumn("tag_color_preference");
  });
}
```

### バリデータ (src/validators/hexColor.ts)

```ts
const HEX_7 = /^#[0-9A-Fa-f]{6}$/;

export function isValidHex7(input: unknown): input is string {
  return typeof input === "string" && HEX_7.test(input);
}
```

### コントローラ拡張 (src/controllers/userController.ts の update を拡張)

```ts
import { isValidHex7 } from "../validators/hexColor";

export async function update(req, res) {
  const userId = req.auth.userId;
  const patch: Record<string, unknown> = {};

  if (req.body.tag_color_preference !== undefined) {
    if (!isValidHex7(req.body.tag_color_preference)) {
      return res.status(400).json({ error: "tag_color_preference must be #RRGGBB" });
    }
    patch.tag_color_preference = req.body.tag_color_preference;
  }

  const updated = await db("users").where({ id: userId }).update(patch).returning("*");
  return res.status(200).json(updated[0]);
}
```

### integration テスト (test/integration/users.test.ts)

```ts
import { describe, it, expect } from "vitest";
import request from "supertest";
import { app } from "../../src/app";
import { seedUser, authHeader } from "../helpers";

describe("PATCH /users/me — tag_color_preference", () => {
  it("200: hex 7 文字を受理し、User 全体を返す", async () => {
    const user = await seedUser();
    const res = await request(app)
      .patch("/users/me")
      .set(authHeader(user))
      .send({ tag_color_preference: "#7BD8B5" });
    expect(res.status).toBe(200);
    expect(res.body.tag_color_preference).toBe("#7BD8B5");
  });

  it("400: hex 7 文字以外は弾く", async () => {
    const user = await seedUser();
    const res = await request(app)
      .patch("/users/me")
      .set(authHeader(user))
      .send({ tag_color_preference: "not-a-color" });
    expect(res.status).toBe(400);
  });

  it("401: 未認証は弾く", async () => {
    const res = await request(app)
      .patch("/users/me")
      .send({ tag_color_preference: "#F4A45D" });
    expect(res.status).toBe(401);
  });
});
```

雛形は最小構成です。手元の認証スタック (JWT / セッション / 独自 middleware) に合わせて `authHeader` の中身を差し替えてください。

## トラブルシューティング

- **AI 自走が 3 ステップ目で何度も止まる**: 設計メモの 8 行が曖昧な可能性。「インターフェース」と「データモデル」の行を 1 文字ずつ読み直してください。「いい感じに」「適切に」が混じっていると未決が増えます
- **テスト pass 率が 90% を切る**: 生成テストの seed が不安定なケース。`beforeEach` で DB をリセットしているか、`afterEach` で truncate しているか確認してください
- **PR 下書きが 24 行を超える**: Claude Code に「24 行に圧縮し直して」と 1 行返すと収まります。逆に 12 行未満になる場合は 8 行設計メモが浅すぎる可能性があります

---

[← episode-03-collaboration ハブに戻る](./)
