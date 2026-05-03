# cc-works-cl notes

note 連載「副業サラリーマンのためのAI使い分けマップ」の **取材レポート全文置き場**。
本サイトは GitHub Pages (Jekyll + jekyll-theme-minimal) でホストする補足ハブです。

公開 URL (GitHub Pages 有効化後): `https://cc-works-cl.github.io/notes/`

## ディレクトリ

```
.
├── _config.yml             # Jekyll 設定 (theme: jekyll-theme-minimal)
├── index.md                # トップページ
├── episode-02-agent/       # 第2話 補足
│   ├── index.md
│   └── *.md                # 取材レポート全文 8本
├── Gemfile                 # GitHub Pages 互換
└── .gitignore
```

## 第2話以降のレポート追加手順

1. note 連載の取材完了後、本リポに `episode-NN-<slug>/` ディレクトリを作る
2. 取材レポート md を配置 (citeturn / entity[...] は除去済みの状態で)
3. `episode-NN-<slug>/index.md` にレポート目次を書く
4. ルート `index.md` のシリーズ表に行を追加
5. `git push` で自動デプロイ

## ローカル確認 (任意)

```bash
bundle install
bundle exec jekyll serve
# → http://localhost:4000
```

## デプロイ

`main` ブランチへ push するだけ。GitHub Pages 設定で:

- Settings → Pages
- Source: Deploy from a branch
- Branch: `main` / root (`/`)

## 運用ルール

- レポート md には ChatGPT 内部の `citeturn〇〇`、`entity["...","name","..."]` などの機械タグを残さない (note 公開と同じ規律)
- 著者識別 alias は `cc-works-cl` (note の `cc_works_cl` と一致)
- 本リポと note 以外の場所 (個人 GitHub `clshinji` 等) との紐付けはしない
