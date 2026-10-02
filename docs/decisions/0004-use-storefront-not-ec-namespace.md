# 0004: 独自の顧客向け機能に `Storefront` 名前空間を使う

- 状態: 採用
- 決定日: 2026-10-01

## 状況

Solidus本体のEngineは `Spree::` 名前空間を使います。Pastbuzz独自の顧客向けEC機能に別のRuby名前空間を設ける場合、`Ec::` は英語として不自然で、顧客向けと管理側のどちらを指すかも明確ではありません。

## 決定

- 独自のEC名前空間として `Ec::` / `:ec` は採用しません。
- Solidus Engineは `Spree::` のままマウントします。Engine全体を `namespace :ec` で包みません。
- Solidusの標準ストアフロントを導入する場合は、まずStarter Frontendの `Spree::` 構成を維持します。
- 自社独自の顧客向け機能に専用名前空間が必要な場合は `Storefront::` を使います。
- 管理側の拡張はSolidusの `Spree::Admin` の構成に従い、`Storefront::` と混同しません。

## 影響

この決定はRubyの名前空間に関するものです。URLのprefix、トップページ、`/products` や `/cart` のルート所有者は別途決定します。

## 後続の決定

2026-10-02に、トップページ `/` はStarter Frontend、会員画面は `/my`、自社APIは `/api/v1/pastbuzz/...`、独自管理機能は `/admin/pastbuzz/...` とする方針を決定しました。この名前空間の決定は引き続き維持します。現在のURL方針と統合時の確認事項は[Solidusの決定事項と課題](../solidus-decisions.md#s2-urlとコード上の名前空間)を参照してください。