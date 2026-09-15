# zenn-content

Zenn（https://zenn.dev/）に投稿する記事の置き場。GitHub 連携で `main` ブランチの `articles/` が自動で反映される。

- 記事の下書きと投稿の決まりは、別リポジトリ claude-self の `docs/articles/README.md` にある。
- `published: false` のうちは Zenn 上で下書き扱い。公開するときに `true` にして push する。
- 手元で確認するときは `npx zenn preview`。
