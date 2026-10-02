# Solidusのカスタマイズ

この文書は、PastbuzzでSolidus 4.7系を拡張するときの基本方針を定めます。Solidus本体のGemを直接変更せず、ホストアプリケーション側で拡張します。

## 優先順位

Solidusを拡張する場合は、次の順に検討します。

1. Solidusの標準機能や設定で実現する
2. 公開された拡張ポイント、イベント、差し替え可能なサービスクラスを使う
3. 新しいアプリケーション機能を追加し、Solidusの内部実装を変更せずに要件を満たす
4. 上記で実現できない場合に限り、Solidusクラスをoverrideする

OverrideはSolidus内部の実装に依存するため、Solidus更新時の確認と、自社側の回帰テストが必要です。

## 名前空間と責務

アプリケーション独自の顧客向け機能は、アプリケーションの名前空間に実装します。たとえば `Storefront::CampaignsController` は自社のストアフロント機能です。

Solidusクラスを拡張するoverrideは、対象クラスを変名したり `Storefront::` に移したりしません。対象は `::Spree::Product` や `::Spree::Admin::...Controller` など、Solidusが定義するクラスのままです。ルーティングの名前空間やURL設計と、Rubyクラスへのoverrideは別の仕組みです。

## Overrideの配置と読み込み

ホストアプリケーションのoverrideは `app/overrides/` に配置します。このディレクトリは通常のZeitwerk自動読み込みから除外し、Railsの `to_prepare` でファイルをロードします。これにより、起動時と開発環境でのリロード時にSolidusクラスへ拡張を適用できます。

`config/application.rb` に以下の設定を追加します。既存の `config.load_defaults 8.1` はそのまま維持します。

```ruby
module Pastbuzz
  class Application < Rails::Application
    config.load_defaults 8.1

    overrides = Rails.root.join("app/overrides").to_s
    Rails.autoloaders.main.ignore(overrides)

    config.to_prepare do
      Dir.glob("#{overrides}/**/*.rb").sort.each do |override|
        load override
      end
    end
  end
end
```

このローダーは、アプリケーション側で明示的に設定するものです。`app/overrides/` ディレクトリを作るだけではRubyファイルは適用されません。Rails 8.1ではZeitwerkが標準のため、`config.autoloader = :zeitwerk` は追加しません。

## Module#prepend

Solidusクラスの振る舞いを変更する必要がある場合は、クラスを直接再オープンするより、名前の付いたモジュールを `prepend` する方法を基本とします。元の処理を維持する場合は `super` を呼び、Solidusが期待する引数、戻り値、副作用を保ちます。

```ruby
# app/overrides/pastbuzz/spree/product/hide_products_override.rb
module Pastbuzz
  module Spree
    module Product
      module HideProductsOverride
        def available?
          ENV["HIDE_ALL_PRODUCTS"] != "1" && super
        end

        ::Spree::Product.prepend(self)
      end
    end
  end
end
```

この例は配置と適用方法を示すものです。実際のoverrideでは、変更対象のメソッドがSolidusの公開APIとして安定しているか、すべての呼び出し箇所で振る舞いを保てるかを先に確認します。`class_eval` を一律に禁止するのではなく、標準の拡張ポイントや `prepend` で安全に対応できない特殊なケースでは、理由を記録して選択します。

## 注文金額・ポイント

`Spree::Order#total` の戻り値だけを上書きしてポイント相当額を差し引く実装は避けます。注文合計は調整額、税、支払い、チェックアウト処理などと整合する必要があります。ポイント利用は、要件に応じてSolidusのAdjustment、Promotion、Store Creditなど既存の仕組みを調査し、適切なドメインモデルと残高更新を含めて設計します。

## テストと更新

Overrideを追加したら、少なくとも次をテストします。

- overrideが意図したSolidusクラスに適用されること
- `super` を含む既存動作と、自社独自の振る舞い
- 注文や決済など、Solidus全体の利用フローにおける結果
- 開発時の再読み込み後も変更が正しく適用されること

Solidusを更新するときは、override対象のメソッド、引数、戻り値、呼び出し元が変わっていないかを再確認します。Solidus内部のテストを丸ごと複製するのではなく、自社拡張の契約と重要な利用フローを検証します。

## 拡張Gemとの違い

`SolidusSupport::EngineExtensions` が提供する `app/decorators` や `lib/decorators/backend` などの仕組みは、Solidus拡張Gem向けです。ホストアプリケーションの `app/overrides` に配置する方法と混同しないでください。

## 参考資料

- [Solidus 4.7: Customizing the core](https://guides.solidus.io/customization/customizing-the-core)
- [Rails Guides: Overriding models and controllers](https://guides.rubyonrails.org/engines.html#overriding-models-and-controllers)
- [Making Solidus Customizations More Resilient](https://supergood.software/making-solidus-customizations-more-resilient/)