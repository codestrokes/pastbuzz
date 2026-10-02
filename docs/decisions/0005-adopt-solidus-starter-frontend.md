# 0005: Solidus Starter Frontendを採用する

- 状態: 採用
- 決定日: 2026-10-01

## 状況

Solidus Core、Backend、APIは導入済みですが、ストアフロントの画面・ルートはまだありません。独自Rails画面だけでストアフロントを構築する案と、Solidus Starter Frontendを利用する案を検討しました。

## 決定

Solidus Starter Frontend一式をPastbuzzのストアフロントの土台として採用します。必要な画面・ルート・アセット等を最初から選別して捨てるのではなく、一式を生成して動く基準として維持します。Starter Frontendを使わず、ストアフロントをすべて独自実装する案は採りません。

Auth0やMinitestなど既存方針と衝突する認証・テスト関連ファイルや依存は、ストアフロント本体を部分採用に戻すのではなく、それぞれの境界で個別に調整します。この決定は、まだStarter Frontendを生成・導入したことを意味しません。

## 実施方針

Solidus 4.7対応のStarter Frontendを作業ブランチに一式生成し、コントローラー、ビュー、ルート、アセット、Gemfile、認証、テストの差分を確認して本体へ取り込みます。認証とテストは、それぞれの決定記録に沿って調整します。

このアプリで利用できる生成コマンドは次のとおりです。

```sh
bin/rails generate solidus:install --frontend=starter
```

既存アプリへ実行する場合は、上書き・Gem追加・migrationや設定変更の内容を事前に確認します。

## 影響

- Starter Frontendのビュー、コントローラー、ルート、アセット、テスト等はアプリへコピーされ、Gem更新だけでは自動更新されません。
- テンプレートが追加するDevise関連機能、Tailwind CSS、RSpec等を、PastbuzzのAuth0・Minitest方針と照合します。
- URLと名前空間は別途決定し、Solidus Engineの `Spree::` 構成を `Ec::` に移し替えません。