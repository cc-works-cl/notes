---
title: 4 工程の実走ログ — 第3話【協働編】
---

# 4 工程の実走ログ — Claude Code 1 本で「設計 → 実装 → レビュー → PR」を回す

note 本編 (第3話) で言及した数値・プロンプト・PR 下書きの **生素材** をここに置きます。題材は架空タスクメッシュ案件の `User.tag_color_preference` カラム追加 1 タスク、観察ベースは手元の個人プロジェクトで同等構造のタスクを 3 本回した範囲です。

## 検証環境

- スタック: Node.js 20 + Express 4 + Knex 3 + Postgres 16 + Vitest 1 + Supertest
- Claude Code: Claude Max プラン、モデル Claude Opus 4.7 (1M context)
- ホスト: macOS (Apple Silicon)
- 1 タスク所要時間の中央値: 実時間 30 分 / AI 自走 1 時間 50 分 / コミット 7 本 / テスト pass 率 92%

## フェーズ 1: 設計 — ノートの 8 行

紙のノートに 4 点 (インターフェース / データモデル / 依存追加 / テスト粒度) を書き出した結果がこれです。本編 h2「設計は人間が握る」で抜粋した 8 行と同一です。

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

書き出しに要した時間は 10 分 (8〜12 分のレンジ)。この 8 行が、後段のセルフレビューと当日確認 5 項目の「整合」基準になります。

## フェーズ 2: 実装 — /plan に出した指示と返ってきた 6 ステップ

Claude Code のセッションを `claude` で開き、上の 8 行をそのまま貼り付けたあと、次の 1 行を続けて打ちます。

```
この設計メモに沿って計画を 6 ステップ以内で。判断が割れる箇所は「人間に承認を求めます」で止めてください。
```

返ってきた 6 ステップは次の構造でした (数十秒で返ってきます)。

1. マイグレーション作成 — `users.tag_color_preference VARCHAR(7) NOT NULL DEFAULT '#F4A45D'` を ADD COLUMN、DOWN で DROP COLUMN
2. `User` モデルに `tag_color_preference` prop を追加
3. **(承認待ち)** PATCH handler 追加 — 既存の `userController.update` を拡張するか、新規 `userController.updatePreference` を作るか
4. バリデーションヘルパ — hex 7 文字の正規表現 `^#[0-9A-Fa-f]{6}$` で 422 を返す
5. integration テスト追加 — supertest で `PATCH /users/me` の 200 / 422 / 401 の 3 ケース
6. ローカル CI を回す — `pnpm test` + `pnpm lint` で緑を確認

3 ステップ目だけ「人間判断」で止まりました。私は 1 行で「**既存の `update` を拡張、新規ハンドラは作らない**」と返し、残り 5 ステップは「進めて」とだけ書きました。

ここから先は AI 自走です。並走させた SubAgent 構成は 2 種類:

- **テスト並行 SubAgent**: 実装メインの裏で integration テスト 3 ケースを先に書かせ、実装側がテストに通る形で fix されるよう順序を組ませる
- **セルフレビュー SubAgent**: 実装完了直後に diff を読ませ、観点 (認可 / null チェック) を 1 行で渡して指摘を返させる

寝る前の最後の作業は `claude` を tmux の中で動かしたまま画面を消すだけ。AI 自走時間は 1 時間 50 分 (1 時間 30 分〜2 時間 10 分のレンジ) で、ブランチが緑のまま静止するのを翌朝確認します。

### コミット 7 本の構造

`git log --oneline --reverse main..feat/user-tag-color-preference` 相当の構造を抜粋すると次のとおりです (具体 SHA はリポジトリごとに変わるので省きます、メッセージのみ)。

```
feat(migration): add tag_color_preference column to users table
feat(model): add tag_color_preference prop to User
feat(controller): extend userController.update to accept tag_color_preference
feat(validation): add hex-color validator (7 chars, #RRGGBB)
test(users): cover PATCH /users/me 200/422/401 with supertest
chore(test): seed user with default tag color before each integration case
chore(ci): run pnpm test + lint locally to confirm 92% pass before push
```

層別ボトムアップ (migration → model → controller → validation → test → seed → CI) で 1 コミット = 1 関心事の粒度。`git revert <SHA>` で単位ごとに巻き戻せます。

## フェーズ 3: レビュー — セルフレビュー指示の生プロンプト

朝、実装が終わったあとに Claude Code 自身へ投げたセルフレビュー指示の全文がこれです。

```
今書いた diff を、以下 3 点の観点で 1 回セルフレビューしてください。
- 設計メモ (ノート 8 行) との整合 — 外れている箇所があれば指摘
- 副作用 — git diff --stat で想定外のファイルが触られていないか
- テスト — 既存テストへの reg、生成テストの assertion の弱さ
判断が割れる箇所は「未決」ラベルで挙げて、私の判断を待ってください。
```

返ってきた未決事項 3 件と、私が判断した内容です。

### 未決 1: `tag_color_preference` カラムの default 値

**Claude Code の指摘**: 「`NOT NULL DEFAULT '#F4A45D'` を入れるか、`NULL` 許容にして UI 側で default を表示するか、設計メモには default 値はあるが NULL 許容の方針が書かれていない」

**人間判断 (30 秒)**: 設計メモに `default '#F4A45D', nullable false` と書いてあるので、**`NOT NULL DEFAULT '#F4A45D'` を採用**。理由は既存ユーザの行を移行時に埋める運用を回避できるため。

### 未決 2: PATCH のレスポンス形状

**Claude Code の指摘**: 「PATCH /users/me のレスポンスとして、`User` オブジェクト全体を返すか、更新した `tag_color_preference` 1 フィールドだけ返すか、既存 API の規約が `users.update` で `User` 全体を返している」

**人間判断 (30 秒)**: 「**既存の API 規約に合わせて**」と 1 行返し、`User` 全体を返す形に。これは Claude Code 側が既存規約を読み取って合わせ込む形で実装が落ちました。

### 未決 3: hex 7 文字以外のバリデーション失敗時の HTTP ステータス

**Claude Code の指摘**: 「hex 7 文字以外を弾くときに 400 (Bad Request) と 422 (Unprocessable Entity) のどちらを返すか、既存の `users.update` は 400 を使っている」

**人間判断 (30 秒)**: 「**既存の API 規約に合わせて**」と 1 行返し、400 で揃える形に (note 本編では 422 と書きましたが、本サイトの再現リポでは既存規約に合わせて 400 で揃えました)。

3 件の未決のうち、手を動かしたのは未決 1 だけ。残り 2 件は「既存規約合わせ」の 1 行で済み、4 分後にセルフレビューが「3 件すべて解消」と返してきました。

### 副作用範囲の確認

朝に手元で `git diff --stat` を眺めた結果です。

```
db/migrations/20260528_add_tag_color_preference.ts | 18 ++++++
src/models/user.ts                                  | 14 +++++-
src/controllers/userController.ts                   | 22 +++++++-
src/validators/hexColor.ts                          | 11 ++++
test/integration/users.test.ts                      | 77 +++++++++++++++++++++
5 files changed, 142 insertions(+), 8 deletions(-)
```

触ったファイル 5 本、追加 142 行、削除 8 行。設計メモの「影響範囲」(User モデル + migration 1 本 + handler 1 本) と一致しています。

## フェーズ 4: PR 下書き — 生成指示と返ってきた 24 行

PR 下書きを Claude Code に書かせる指示の生プロンプトはこれです。

```
このブランチの git log と diff から PR 説明文を 24 行以内で書いてください。
構造は タイトル / 背景 / 変更点 / 動作確認 / レビュー観点 / 関連 Issue。
背景と動作確認とレビュー観点は箇条書きで、私が後から 1 行ずつ書き直す前提。
タイトルは Conventional Commits (feat: ◯◯ のように種別 prefix を付ける書式) で。
```

返ってきた 24 行の下書きと、人間が直した 3 行の before / after をここに置きます。

### 下書き (Claude Code 生成、24 行)

```markdown
## feat(user): add tag_color_preference column and PATCH /users/me endpoint

### 背景
- `User` モデルにタグ色の保存先を追加するため、`tag_color_preference` カラムを足す
- PATCH /users/me で更新可能にし、UI 側のタグ色保存ニーズに対応する
- 既存の `users.update` API の規約を踏襲する形で実装

### 変更点
- db/migrations: `users.tag_color_preference VARCHAR(7) NOT NULL DEFAULT '#F4A45D'` を ADD / DROP の両方で実装
- src/models/user.ts: `tag_color_preference` prop を追加
- src/controllers/userController.ts: 既存の `update` を拡張して `tag_color_preference` を受理
- src/validators/hexColor.ts: hex 7 文字の正規表現バリデータを新設、不正値は 400 を返す
- test/integration/users.test.ts: PATCH /users/me の 200 / 400 / 401 の 3 ケースを supertest で追加

### 動作確認
- `pnpm prisma migrate dev` でマイグレーション適用、`pnpm test` で 92% pass
- ローカルで `curl -X PATCH http://localhost:3000/users/me ...` を叩いて 200 が返ることを確認
- DB に値が反映されることを `psql` で確認

### レビュー観点
- 既存の認可 middleware が新カラムにも効いているか
- hex バリデーションの 400 ステータスが既存規約と整合しているか
- migration の DOWN が DROP COLUMN になっているか

### 関連 Issue
- Closes #(架空 Issue 番号、再現時に差し替え)
```

### 人間が直した 3 行 (before → after)

PR テンプレ通り、背景・動作確認・レビュー観点を 1 行ずつ書き直しました。

**背景 (1 行に圧縮)**:

```diff
- - `User` モデルにタグ色の保存先を追加するため、`tag_color_preference` カラムを足す
- - PATCH /users/me で更新可能にし、UI 側のタグ色保存ニーズに対応する
- - 既存の `users.update` API の規約を踏襲する形で実装
+ - タグ色のユーザー別保存はタグ機能の仕上げに必要で、本実装でユーザーごとの個別化が完成する
```

**動作確認 (再現可能な形に整理)**:

```diff
- - `pnpm prisma migrate dev` でマイグレーション適用、`pnpm test` で 92% pass
- - ローカルで `curl -X PATCH http://localhost:3000/users/me ...` を叩いて 200 が返ることを確認
- - DB に値が反映されることを `psql` で確認
+ - `pnpm dev` 起動 → `curl -X PATCH http://localhost:3000/users/me -H 'Authorization: Bearer <token>' -d '{"tag_color_preference":"#F4A45D"}' -i` → `psql -c "select tag_color_preference from users where id='<uuid>'"` で値の反映を確認
```

**レビュー観点 (1 点に絞って指名)**:

```diff
- - 既存の認可 middleware が新カラムにも効いているか
- - hex バリデーションの 400 ステータスが既存規約と整合しているか
- - migration の DOWN が DROP COLUMN になっているか
+ - `User` モデル拡張なので、既存の認可 middleware が新カラムにも効いているかの目線が欲しい
```

下書き 24 行のうち、書き直したのは 3 ブロックだけ。残りはそのまま採用しました。PR 提出までに使った人間の時間は 5〜8 分 (中央値 5 分)、Approve ボタンを押した時刻は 23:58 でした。

## 数値の生計測ログ (3 本回した範囲)

本編「コスト感と所要時間」の数値の出所です。

| run | 設計 | AI 自走 | レビュー | PR | 合計実時間 | コミット数 | pass 率 |
|---|---|---|---|---|---|---|---|
| run-1 | 12 分 | 2 時間 10 分 | 18 分 | 8 分 | 38 分 | 8 本 | 90% |
| run-2 | 10 分 | 1 時間 50 分 | 15 分 | 5 分 | 30 分 | 7 本 | 92% |
| run-3 | 8 分 | 1 時間 30 分 | 12 分 | 4 分 | 24 分 | 5 本 | 95% |
| **中央値** | **10 分** | **1 時間 50 分** | **15 分** | **5 分** | **30 分** | **7 本** | **92%** |

3 本とも同一スタック (Node + Express + Knex + Postgres + Vitest)、観点を変えた 3 つの 1 カラム追加タスクで観察。Claude Code Opus 4.7 利用、tmux + macOS の組み合わせ。

## 観察コメント

- **設計フェーズの 10 分が効く**: ノート 8 行を書き出す前に Claude Code を起動した別タスクでは、6 ステップ計画のうち 4 ステップで「人間判断」が入り、合計実時間が 50 分を超えました。8 行を先に書く効果は、後段の自走と未決事項の少なさに直結します
- **未決事項 3 件は安定**: 観察した 3 本のすべてで未決事項が 2〜4 件のレンジに収まりました。「default 値 / レスポンス形状 / バリデーションの HTTP ステータス」の組み合わせが繰り返し上がる傾向があります
- **PR 下書き 24 行は人間 3 行で着地する**: 下書きが長すぎても短すぎても直す手間が増える、というのが 3 本回しての感触で、24 行前後が最短ルートでした

## 再現したい方へ

手元のリポで同等構造のタスクを回す手順は [reproduce](./reproduce) に置いてあります。手元に Node + Express + Postgres の最小構成があれば、本サイトの DDL とハンドラの雛形を貼って、上のプロンプト 3 種類を順に Claude Code に投げるだけで、同じ 4 工程を再現できます。

---

[← episode-03-collaboration ハブに戻る](./)
