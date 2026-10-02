# Pastbuzz

Rails 8.1 / Solidus 4.7 を使ったストアフロントアプリケーションです。初期の販売・配送対象は日本国内です。

## 開発環境

- Docker
- Visual Studio Code と Dev Containers 拡張機能

リポジトリをDev Containerで開くと、Ruby 3.4.9、MySQL 8.4、System Test用のSeleniumが利用できます。コンテナ作成後にRuby依存GemのインストールとDBセットアップが実行されます。

```sh
bin/rails server
```

アプリケーションは `http://localhost:3000` で確認できます。

```sh
RAILS_ENV=production SECRET_KEY_BASE_DUMMY=1 bin/rails assets:precompile
```

## テスト

```sh
bin/rails test
bin/rails test:system
```

アプリケーションのテストは `test/` 配下のMinitestを使用します。詳細は[テスト方針](docs/testing.md)を参照してください。
