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

SolidusのCore Engineをルート (`/`) にマウントしています。Solidusの標準ストアフロントおよび管理画面のルートは、Solidusの構成に従います。

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

## 検討中の事項

以下はまだ実装上の決定事項ではありません。要件、既存システムとの互換性、運用コストを確認してから個別に決定します。

- 認証プロバイダーと既存会員情報の移行方法
- 会員機能のURLと名前空間の設計
- CSSおよびUIコンポーネントの採用方針
- 旧URLから新URLへのリダイレクト方針
