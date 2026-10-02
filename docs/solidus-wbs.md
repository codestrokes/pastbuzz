# Solidus実装WBS（初版）

Solidus 4.7.1とRails 8.1.4を使うPastbuzzの初期ストアフロント実装計画です。採用済みの方針と未決事項は[Solidusの決定事項と課題](solidus-decisions.md)、overrideの実装方針は[Solidusのカスタマイズ](solidus-customization.md)を参照してください。

このWBSは作業順・成果物・完了条件を示します。税・決済・配送の詳細、作業者、所要時間が未確定のため、日程・工数はまだ見積もりません。ポイント機能は初期リリース対象外とします。

## 前提とスコープ

- Solidus Starter Frontend一式をストアフロントの土台として使います。
- Starter Frontendはまだ導入されていません。生成コードを作業ブランチで確認してから統合します。
- `Ec::` は使いません。Solidusの `Spree::` 構成を維持し、自社独自の顧客向け機能に必要なら `Storefront::` を使います。
- 初期販売・配送地域は日本国内のみです。
- ホストアプリのテストはMinitestに統一します。生成されるRSpec資産は仕様の参考として扱います。
- Auth0の採用前提はあるものの、Solidusユーザーとの接続方法は未決定です。
- `/admin`、`/api` はSolidusの既存ルートと衝突しないよう統合します。

## WBS

| ID | 作業 | 成果物・完了条件 | 依存・状態 |
|---|---|---|---|
| 1.1 | 導入前の技術確認 | Starter Frontend v4.7と現在のRails/Solidusの互換性、生成ファイル、依存Gem、installerの認証選択挙動を確認 | なし。完了（隔離worktreeで生成・確認） |
| 1.2 | URLと入口の決定 | `/` の担当、会員URL（`/my` 等）、自社API・管理機能のprefix、認証・認可境界を決定。Solidus既存ルートとの重複なし | 1.1。完了（`/`、`/my`、`/api/v1/pastbuzz`、`/admin/pastbuzz`と認証境界を決定。衝突テストは統合時） |
| 1.3 | Auth0とSolidus Userの設計 | Auth0 subjectと `Spree::User` の一意な対応、ログイン/ログアウト、ゲスト注文、管理者認証方針を決定 | 1.1。完了（subject専用列、管理者Devise、ゲスト購入、既存会員の明示連携を決定） |
| 1.4 | 日本国内の販売要件 | JPY、ロケール、消費税・表示、国内住所・配送、決済方法、返品/返金要件を決定 | なし。進行中（国内全国配送・税込表示・軽減税率商品・Stripeカード決済を決定。会計確認と送料・返品条件は残る） |
| 1.5 | ポイント要件 | 付与・利用・失効・取消/返品時の戻し・会計処理を定義し、Solidus上の実現方式を決定 | 1.4。後続フェーズ（初期リリース対象外） |
| 2.1 | Starter Frontendの検証生成 | 作業ブランチまたは隔離した検証環境に公式コマンドで生成し、変更一覧を保存 | 1.1。完了（隔離worktreeで生成） |
| 2.2 | 生成差分のレビュー | Gemfile/lock、認証、RSpec、Tailwind、Sprockets、routes、migration、環境設定の差分と採否を記録 | 2.1。完了（Auth0/Minitestとの衝突を確認） |
| 2.3 | Starter一式のアプリ統合 | ストアフロント一式を土台として取り込み、`/up`、Solidus管理/APIルート、既存アプリ設定を維持 | 1.2、2.2。未着手 |
| 3.1 | アセットのビルド確認 | 公式のTailwind/Sprockets manifest構成で `bin/rails assets:precompile` が成功。ストアフロントCSSが管理画面へ漏れない | 2.3。未着手 |
| 3.2 | 認証統合 | Auth0で認証したユーザーが対応する `Spree::User` として注文に紐づく。Devise依存を残す/置き換える方針が実装と一致 | 1.3、2.3。未着手 |
| 3.3 | テスト統合 | `test/` のMinitestを標準とし、RSpec/Minitestを併設しない。重要なフロント動作をMinitestで検証 | 2.2、2.3。未着手 |
| 4.1 | 日本向け店舗設定 | 決定した通貨、ロケール、税、国内配送、決済、在庫設定を反映し、秘密情報をコードに含めない | 1.4、2.3。未着手 |
| 4.2 | 会員機能との接続 | 会員向け画面/API/管理機能のルートをSolidusルートと衝突させず実装。共有ロジックと入口ごとの認可を分離 | 1.2、3.2。未着手 |
| 4.3 | ポイント連携 | 合意済み要件に従ってポイント台帳・注文調整等を実装。注文金額getterの上書きだけで割引しない | 1.5、3.2、4.1。後続フェーズ（初期リリース対象外） |
| 5.1 | 主要購入フローの検証 | 商品閲覧からカート、チェックアウト、決済、注文完了までの主要経路をMinitest/System Testで検証 | 3.1、3.2、3.3、4.1。未着手 |
| 5.2 | 取消・返金・障害ケースの検証 | 注文取消、Stripe決済失敗・追加認証失敗、Webhook、返金、在庫不足、Auth0失敗時の整合性を確認 | 4.1、5.1。未着手 |
| 5.3 | リリース準備 | 本番相当環境でアセット、DB migration、初期設定、監視、バックアップ、復旧手順を確認 | 5.1、5.2。未着手 |

## 先行して解消するリスク

1. **確認済み:** `--frontend=starter --authentication=custom` でもinstallerはDeviseを選択し、スターターテンプレートがDevise用Gem・migration・画面を追加します。Auth0採用時の置換範囲を設計してから本体へ統合します。
2. **確認済み:** スターターはRSpec関連Gemと `spec/` を追加します。Minitestのみを維持し、RSpecをホストアプリのテストに残さないよう生成差分から除外します。
3. **一部確認済み:** 検証生成では `/`、`/products`、`/cart` とSolidusの `/admin`・`/api`、`/up` が共存しました。本体統合時にも `bin/rails routes` とrequest testで回帰確認します。
4. **確認済み:** `--auto-accept` は決済方法も既定選択し、検証時はPayPal連携を追加しました。Starter生成時は `--payment-method=none` を明示し、公式 `solidus_stripe` を別途追加・設定して初期決済をカードのみに制限します。
5. **環境対応済み:** Solidus BackendのSprockets/SassC compressorがTailwind CSSを処理できず失敗することを確認しました。本番CSS compressorを無効にしてNode.jsなしでprecompileが成功し、Tailwind CSSは自身の本番minify機能で圧縮されます。Tailwind以外のSprockets CSSの成果物サイズは別途確認します。
6. 消費税率・軽減税率の商品区分、Stripeアカウント/Webhook設定、送料・返品条件は引き続き確定が必要です。ポイントは初期リリース対象外です。

## 完了条件

- Starter Frontendを基準とする日本向けストアフロントが動作する。
- Auth0ユーザーとSolidus注文所有者の対応が明確で、認証・認可テストが通る。
- Solidus管理画面、API、ストアフロント、自社会員機能のルートが重複しない。
- アセットprecompile、本番相当の主要購入フロー、取消・返金の確認が通る。
- 初期販売・配送対象が日本国内に限定され、決済・税・運用の必要設定が完了している。