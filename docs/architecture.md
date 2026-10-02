# アーキテクチャ

## 概要

Pastbuzzは、Rails 8を基盤とするモノリシックなWebアプリケーションです。アプリケーションコード、SolidusによるEC機能、データベース、バックグラウンド処理を単一のリポジトリで管理します。

## 技術要素

- Ruby on Rails 8.1
- Solidus 4.7
- TrilogyによるMySQL 8.4接続
- Hotwire（Turbo / Stimulus）
- Importmap
- Solid Cache、Solid Queue、Solid Cable
- Puma

データベースはMySQL 8.4を使用します。選定理由は[MySQL 8.4の採用](decisions/0007-use-mysql-84.md)を参照してください。

## 責務分担

### Railsアプリケーション

業務固有の機能、画面、認証・認可、外部サービス連携、自社固有のデータモデルを実装します。

### Solidus

商品、価格、在庫、カート、注文、決済など、ECの基盤機能を提供します。標準機能を優先して利用し、自社要件との差分はアプリケーション側の拡張として実装します。

### ルーティング

SolidusのCore Engineをルート (`/`) にマウントし、Solidusの管理画面とAPIのルートを利用しています。Starter FrontendのストアフロントルートはCore Engineとは別にホストアプリへ生成されます。Starter Frontendは本体へまだ導入されていません。

アプリケーション固有のルートは `config/routes.rb` に追加します。Solidus標準のルートや名前付きルートを、理由なく上書きしない方針です。

## ディレクトリ方針

- `app/models/`: アプリケーション固有のモデルとSolidus拡張
- `app/controllers/`: アプリケーション固有のコントローラー
- `app/javascript/`: StimulusコントローラーなどのJavaScript
- `app/views/`: アプリケーション固有のビュー
- `test/`: Minitestによるテスト
- `config/`: RailsおよびSolidusの設定
- `db/`: スキーマとマイグレーション

## 変更時の原則

1. まずRailsまたはSolidusの標準機能で実現できるか確認する。
2. 拡張が必要な場合は、Gem本体を直接変更せず、アプリケーション側で拡張する。
3. 仕様変更と同時に、影響を受ける自社コードのテストを追加または更新する。
4. この文書は実装の現状と一致するように更新する。

Solidusの拡張方法、overrideのロード設定とテスト方針は[Solidusのカスタマイズ](solidus-customization.md)を参照してください。

Solidusの導入・認証・ストアフロント・運用に関する未決事項と推奨順序は[Solidusの決定事項と課題](solidus-decisions.md)を参照してください。

実装作業の順序、依存関係、成果物と完了条件は[Solidus実装WBS](solidus-wbs.md)を参照してください。

## 決定済みの実装方針

以下は採用済みの方針であり、まだ本体への実装が完了したことを意味しません。詳細は[Solidusの決定事項と課題](solidus-decisions.md)を参照してください。

- トップページ `/` はStarter Frontendのホーム画面を使います。
- 会員画面は `/my`、自社APIは `/api/v1/pastbuzz/...`、独自管理機能は `/admin/pastbuzz/...` に配置します。自社APIはSolidus APIとは別のホストアプリ側endpointとし、認証・認可を独立して定義します。共通の業務処理のみモデルやサービスで共有します。Rubyの名前空間とURL prefixは分けて設計し、独自の顧客向け機能には必要に応じて `Storefront::` を使います。
- 会員認証はAuth0を使い、subjectを `Spree::User.auth0_subject` の専用一意列で管理します。管理者認証はSolidus/Deviseを維持します。
- 既存会員は元の認証方式で再認証した後にAuth0へ明示連携し、メールアドレス一致だけでは自動連携しません。ゲスト購入は許可します。
- ストアフロントはStarter Frontendの公式Tailwind・Sprockets構成を使います。

## 検討中の事項

以下はまだ実装上の決定事項ではありません。要件、既存システムとの互換性、運用コストを確認してから個別に決定します。

- Auth0連携の具体的な実装手順と既存会員の明示連携フロー
- Auth0 issuerを単一に限定する前提の確定、Auth0/Deviseセッション・logout分離、管理者権限保有会員の境界、メール衝突時とゲストカート引継ぎの振る舞い
- Stripeのcapture時期、追加認証中断・再試行・二重決済防止、Webhookの署名検証・重複/遅延/順序逆転、注文状態不一致時の照合・復旧、一部返金の責務分担
- 税率混在注文の税額集計・端数処理、送料/値引きの税務上の扱い、一部返品時の税額・返金額、配送温度帯の要否
- 会員機能の具体的な画面/APIと、決定済みprefix配下でのルート・認可の実装
- Tailwind以外のCSSの圧縮方式と、追加UIコンポーネントの採用方針
- 旧URLから新URLへのリダイレクト方針
