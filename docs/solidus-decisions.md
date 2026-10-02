# Solidusの決定事項と課題

この文書は、PastbuzzでSolidusを導入・拡張する際に決めることを一覧し、判断のたたき台を示します。ここにある推奨は決定済み仕様ではありません。決定した項目は必要に応じて `docs/decisions/` に個別の記録を追加し、この一覧の状態を更新します。

確認日: 2026-10-01

## 現在の状態

- `solidus` 4.7.1を利用しています。依存関係にはSolidus Core、Backend、APIなどが含まれます。
- `Spree::Core::Engine` を `/` にマウントしており、`/admin/...` と `/api/...` のルートがあります。
- Solidus Starter Frontendの採用は決定していますが、まだ生成・導入されていません。現時点で `/products` や `/cart` のストアフロント画面はありません。
- アプリのルートには `/up` のhealth checkがあります。トップページの所有者と振る舞いは未決定です。
- ホストアプリのテストはMinitestです。Solidus Starter FrontendのテンプレートはDevise関連機能、Tailwind CSS、RSpec等を追加するため、導入前に差分を確認する必要があります。
- Auth0、ポイント連携、ストアフロント用のアプリコードはまだ導入されていません。

## 優先して決める項目

### S1. ストアフロントをどう用意するか

- 状態: Starter Frontend一式を土台にする方針は決定。既存方針との統合内容は未確認
- 決定: Solidus Starter Frontendをストアフロントの土台として使い、必要な画面・ルート・アセット等を選別して捨てるのではなく、まず一式を生成して動く基準として維持します。独自実装のみで置き換える案は採りません。
- 保留: Auth0、Minitestなど既存方針と衝突する認証・テスト関連ファイルや依存をどう置き換えるかは、該当項目で決めます。これはストアフロント本体を部分採用に戻す意味ではありません。
- 推奨: Solidus 4.7用Starter Frontendを作業ブランチで生成し、生成ファイル・Gemfile・ルート・アセット・テストを確認して取り込みます。既存ファイルの上書きや認証方式の置換は差分を確認しながら行います。
- 理由: Starter FrontendはGemとして画面を提供するのではなく、テンプレートでコントローラー、ビュー、ルート、アセット、テスト等をアプリへコピーします。生成後のコードはテンプレート更新で自動追従しません。

### S2. URLとコード上の名前空間

- 状態: 一部決定。URL構成とトップページは未決定
- 対象: `/`、`/products`、`/cart`、会員ページ、独自キャンペーンなど
- 決定: 独自のEC名前空間に `Ec` / `:ec` は採用しません。英語で不自然で、顧客向け機能と管理側のどちらを指すかも曖昧なためです。SolidusのEngineは `Spree::` のままマウントし、自社独自の顧客向け機能に名前空間が必要な場合は `Storefront::` を使います。
- 推奨: Solidus Engineのマウントは `mount Spree::Core::Engine, at: "/"` のままにし、`namespace :ec` でEngine全体を包まないでください。Starter Frontendを採用する場合は、まずその `config/routes/storefront.rb` と `Spree::` のコントローラー・ビュー構成を維持します。自社独自の顧客向け機能に限って `Storefront::` を使うのは妥当です。
- ルート衝突に注意: `/admin/...` と `/api/...` はすでにSolidusが使っています。自社の管理・APIルートを同じprefixへ追加する場合は、Solidusのルート拡張として明示的に統合するか、Solidusの既存route name/path/actionと重ならない専用subpathを選びます。`/api/v1/pastbuzz/...` や `/admin/pastbuzz/...` は候補例であり、採用前に `bin/rails routes` とrequest testで衝突を確認します。
- Controllerのコード名前空間はURL prefixと同じにする必要はありません。会員向け画面、API、管理画面は入口ごとの認証・認可を分け、共有する会員ドメインロジックはモデルやサービスへ置きます。
- 会員ページ: 以前の検討では、機能が多岐にわたる会員領域のURLとして `/dashboard` より `/my` が自然、という意見で一致しています。ただし、正式決定とリダイレクト方針は未記録です。
- 要確認: `/` のトップページをStarter Frontend、自社トップ、またはリダイレクトのどれが担当するかを決め、Solidusのルートとの優先順位をルートテストで固定します。

### S3. Auth0とSolidusユーザーの対応

- 状態: 未決定。認証実装前に決める
- 論点: Auth0の識別子とローカルの `Spree::User` の対応、既存ユーザーの連携、ログアウト、権限、ゲスト注文、アカウント重複時の扱い
- 推奨: Auth0の安定したsubjectをローカルユーザーへ一意に結び付け、注文所有者として使うローカルレコードを明確にします。メールアドレスだけを恒久的な識別子にしないでください。
- 注意: Starter Frontendは `solidus_auth_devise` とDeviseベースの画面・ルートを追加します。Auth0採用時に見た目のログイン画面だけを削除して終わりとは限りません。Devise gemを外す前に、`Spree::User` の認証機能、Solidusが利用する認証API、管理者認証、パスワードリセット、既存注文との関連を確認します。

### S4. ポイントと注文・決済の関係

- 状態: 未決定。ポイント要件を定義してから設計
- 決めること: 付与・利用率、失効、利用上限、併用条件、注文取消・返品時の戻し、ゲスト注文、会計上の扱い
- 推奨: ポイントを注文金額の表示値から直接差し引くのではなく、仕様に応じてSolidusのAdjustment、Promotion、Store Credit等の既存モデルと整合する方式を選びます。独自台帳が必要なら、残高と履歴の整合性・冪等性も定義します。
- 禁止する近道: `Spree::Order#total` のoverrideだけで割引を表現しないこと。税、調整、支払額、返金との不整合を招く可能性があります。

### S5. CSS・アセットとTailwind

- 状態: Starter Frontendの公式Tailwind・Sprockets構成を使う。既存アプリへの適用結果は未確認
- 現状: Rails 8.1自体はTailwindを必須・標準同梱していません。現在のGemfileにもTailwindはありません。Propshaftは外しており、Solidus Backendが使うSprocketsが有効です。
- 決定: Starter Frontendを採用するため、スターターが提供するTailwind構成を使います。Solidus BackendとStarter Frontendはそれぞれのmanifestをprecompile登録する構成です。
- 推奨: 公式の生成設定を維持し、既存アプリへの導入後に `bin/rails assets:precompile` と画面表示を確認します。Tailwind CSSはストアフロント側で読み込み、Solidus管理画面へ意図せず適用されないことを確認します。Tailwind Plusのコンポーネントを使う場合は、ライセンス、Tailwindのメジャーバージョン、ビルド対象パスも確認します。
- 注意: Starter Frontendのv4.7テンプレートは `tailwindcss-rails ~> 3.0` を指定しています。「Solidus 4.3のTailwindテーマ」はTailwind CSS 4.3を意味しません。

### S6. テストフレームワーク

- 状態: Minitest採用済み。Starter FrontendのRSpec資産の扱いは未決定
- 推奨: 既存の `docs/decisions/0001-use-minitest.md` に従い、ホストアプリではRSpecとMinitestを混在させません。Starter FrontendのRSpecテストは仕様の参考にし、重要なストアフロントの振る舞いだけをMinitestで検証します。すべての生成RSpecテストを一対一で移植することは前提にしません。
- 必須テスト: 主要な商品閲覧、カート、チェックアウト、認証から注文所有者への紐付け、ポイント利用をユーザー視点で検証します。

### S7. Solidus管理画面の拡張

- 状態: 未決定。管理機能を追加する際に判断
- 推奨: 管理画面はSolidusの `/admin` と `Spree::Admin` を基本とし、顧客向け `Storefront::` と混同しません。独自機能を `/admin` に追加する場合は、Solidus管理画面の認証・認可・レイアウトに統合したうえで、既存route name/path/actionとの重複を避けます。設定・メニュー・イベント等の公開拡張点を優先し、Controller overrideが必要な場合は[Solidusカスタマイズ方針](solidus-customization.md)に従います。

### S8. 店舗・運用設定

- 状態: 初期対象地域は決定。店舗・税・決済等の具体設定は未決定
- 決定: 初期の販売・配送対象は日本国内のみとします。海外への販売・配送は初期スコープに含めず、必要になった時点で別途判断します。
- 決めること: JPYと日本語ロケール、消費税の計算・表示方法、国内住所と配送地域・送料、利用する決済方法、在庫管理、注文番号、メール送信、返品・返金方針
- 推奨: 日本国内の販売・会計要件に合わせて設定し、テスト環境で注文から取消・返金まで確認してから本番化します。税務・適格請求書等の要件は専門家または会計担当に確認します。鍵や決済情報はソースに含めず、環境またはRails Credentials等で管理します。

### S9. テンプレート更新とSolidusアップグレード

- 状態: 運用手順未決定
- 推奨: Starter Frontendのコピー済みコードはGem更新だけでは自動更新されない前提で扱います。Solidus更新時に、対応するStarter Frontendブランチとの差分、独自変更、セキュリティ修正、主要System Testを確認する手順を用意します。

## 推奨する決定順

1. ストアフロントの調達方針と生成差分を確認する（S1）
2. URL、トップページ、会員ページの境界を決める（S2）
3. Auth0と `Spree::User` の対応を決める（S3）
4. 店舗設定とポイントの業務仕様を整理する（S4、S8）
5. アセットとテストの統合方針を確定する（S5、S6）
6. 管理画面の拡張と更新手順を必要に応じて決める（S7、S9）

## 参考資料

- [Solidus 4.7 Starter Frontend](https://github.com/solidusio/solidus_starter_frontend/tree/v4.7)
- [Solidus Starter Frontend v4.7 template](https://github.com/solidusio/solidus_starter_frontend/blob/v4.7/template.rb)
- [Solidus 4.7: Customizing the core](https://guides.solidus.io/customization/customizing-the-core)
- [Solidus Starter Frontendの紹介](https://solidus.io/blog/getting-started-with-solidus-starter-frontend)