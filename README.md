# これはなに？

[Zenn.dev/anon](https://zenn.dev/anon)で公開されている記事を管理するリポジトリです。

基本的に[zenn-cli](https://zenn-dev.github.io/zenn-docs-for-developers/guides/zenn-editor/zenn-cli)を使用して記事を管理します。

## zenn-cliのインストール

プロジェクトページで以下を実行

```shell
$ yarn install
```

その他インストール方法はZennによる [Zenn CLIをインストールする](https://zenn.dev/zenn/articles/install-zenn-cli) などを確認してください。


# 記事の作成

以下のコマンドを使用し、テンプレートを生成

```shell
$ yarn new:article
```

記事URLにslugを指定する場合は以下の様にする

```shell
$ yarn new:article --slug 記事のスラッグ --title 記事のタイトル --type tech
```

# 記事のプレビュー

```shell
$ yarn preview
```

