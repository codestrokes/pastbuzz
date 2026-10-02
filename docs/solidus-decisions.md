# Solidusの決定事項と課題

この文書は、PastbuzzでSolidusを導入・拡張する際に決めることを一覧し、判断のたたき台を示します。ここにある推奨は決定済み仕様ではありません。決定した項目は必要に応じて `docs/decisions/` に個別の記録を追加し、この一覧の状態を更新します。

確認日: 2026-10-02

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
- 確認結果: v4.7.1のinstallerはStarter Frontendを選ぶと認証方式を先にDeviseへ決めるため、`--authentication=custom` を指定してもDeviseが選ばれます。Starter templateも `solidus_auth_devise` と認証用migration・画面を追加します。Auth0を使う場合は、スターターのストアフロントを維持しながら認証境界を別途置き換える必要があります。
- 生成時の注意: `--auto-accept` は決済方法も既定選択するため、決済要件が未決定の段階では `--payment-method=none` を明示します。テンプレートはRSpec関連Gemと `spec/` も追加するので、Minitest方針に合わせて採否を確認します。
- 理由: Starter FrontendはGemとして画面を提供するのではなく、テンプレートでコントローラー、ビュー、ルート、アセット、テスト等をアプリへコピーします。生成後のコードはテンプレート更新で自動追従しません。

### S2. URLとコード上の名前空間

- 状態: URL prefixと認証境界は決定済み。実際のルート衝突テストは機能統合時に実施
- 対象: `/`、`/products`、`/cart`、会員ページ、独自キャンペーンなど
- 決定: 独自のEC名前空間に `Ec` / `:ec` は採用しません。英語で不自然で、顧客向け機能と管理側のどちらを指すかも曖昧なためです。SolidusのEngineは `Spree::` のままマウントし、自社独自の顧客向け機能に名前空間が必要な場合は `Storefront::` を使います。
- 推奨: Solidus Engineのマウントは `mount Spree::Core::Engine, at: "/"` のままにし、`namespace :ec` でEngine全体を包まないでください。Starter Frontendを採用する場合は、まずその `config/routes/storefront.rb` と `Spree::` のコントローラー・ビュー構成を維持します。自社独自の顧客向け機能に限って `Storefront::` を使うのは妥当です。
- 決定: トップページ `/` はStarter Frontendのホーム画面を使います。検証生成では `home#index` が `/` を担当し、Solidusの `/admin`・`/api`、`/up` と共存することを確認しました。
- 決定: 会員向け画面のprefixは `/my` とします。注文履歴やプロフィール等の会員機能はこのprefix配下に配置します。
- 決定: 自社独自の会員APIを追加する場合は `/api/v1/pastbuzz/...` を使い、Solidus APIの既存ルートに統合します。追加時には既存route name/path/actionとの重複をテストします。
- 決定: Solidus管理画面の独自機能は `/admin/pastbuzz/...` を使い、Solidus管理者認証・認可の配下に統合します。
- ルート衝突に注意: `/admin/...` と `/api/...` はすでにSolidusが使っています。自社の管理・APIルートを同じprefixへ追加する場合は、Solidusのルート拡張として明示的に統合するか、Solidusの既存route name/path/actionと重ならない専用subpathを選びます。`/api/v1/pastbuzz/...` や `/admin/pastbuzz/...` は候補例であり、採用前に `bin/rails routes` とrequest testで衝突を確認します。
- Controllerのコード名前空間はURL prefixと同じにする必要はありません。会員向け画面、API、管理画面は入口ごとの認証・認可を分け、共有する会員ドメインロジックはモデルやサービスへ置きます。
- 実装時の確認: `/my`、`/api/v1/pastbuzz/...`、`/admin/pastbuzz/...` のroute name/path/actionがSolidus既存ルートと重ならないことを `bin/rails routes` とrequest testで固定します。

### S3. Auth0とSolidusユーザーの対応

- 状態: 決定済み（Auth0会員認証、管理者認証、ゲスト注文、既存会員の連携方針）
- 論点: Auth0の識別子とローカルの `Spree::User` の対応、既存ユーザーの連携、ログアウト、権限、ゲスト注文、アカウント重複時の扱い
- 決定: Auth0の安定した `sub` はメールアドレスと分け、`Spree::User` の専用一意列 `auth0_subject` に保存します。メールアドレスの変更で別会員にならない構成とします。
- 決定: Auth0はストアフロント会員認証に使い、Solidus管理画面（`/admin`）はSolidus/Deviseの管理者認証を維持します。ストアフロント向けDeviseログインを管理者ログインと混同しないよう、ルートと認可を分離します。
- 決定: Auth0未ログインのゲスト購入を許可します。ゲスト注文はSolidusのゲスト注文として扱い、Auth0 subjectのない注文を会員注文として扱いません。
- 決定: 既存のSolidus会員とAuth0 subjectはメールアドレス一致だけでは自動連携しません。既存会員が元の認証方式で再認証した後に明示的に連携します。
- 推奨: Auth0の安定したsubjectをローカルユーザーへ一意に結び付け、注文所有者として使うローカルレコードを明確にします。メールアドレスだけを恒久的な識別子にしないでください。
- 注意: Starter Frontendは `solidus_auth_devise` とDeviseベースの画面・ルートを追加します。Auth0採用時に見た目のログイン画面だけを削除して終わりとは限りません。Devise gemを外す前に、`Spree::User` の認証機能、Solidusが利用する認証API、管理者認証、パスワードリセット、既存注文との関連を確認します。

### S4. ポイントと注文・決済の関係

- 状態: 初期リリースでは後続に延期。要件未定のためポイント利用・台帳は実装しない
- 決定: 初期リリースではポイント付与・利用を提供しません。別フェーズで要件を確定してから設計・実装します。
- 決めること: 付与・利用率、失効、利用上限、併用条件、注文取消・返品時の戻し、ゲスト注文、会計上の扱い
- 推奨: ポイントを注文金額の表示値から直接差し引くのではなく、仕様に応じてSolidusのAdjustment、Promotion、Store Credit等の既存モデルと整合する方式を選びます。独自台帳が必要なら、残高と履歴の整合性・冪等性も定義します。
- 禁止する近道: `Spree::Order#total` のoverrideだけで割引を表現しないこと。税、調整、支払額、返金との不整合を招く可能性があります。

### S5. CSS・アセットとTailwind

- 状態: Starter Frontendの公式Tailwind・Sprockets構成を使う。既存アプリへの適用結果は未確認
- 現状: Rails 8.1自体はTailwindを必須・標準同梱していません。現在のGemfileにもTailwindはありません。Propshaftは外しており、Solidus Backendが使うSprocketsが有効です。
- 決定: Starter Frontendを採用するため、スターターが提供するTailwind構成を使います。Solidus BackendとStarter Frontendはそれぞれのmanifestをprecompile登録する構成です。
- 推奨: 公式の生成設定を維持し、既存アプリへの導入後に `bin/rails assets:precompile` と画面表示を確認します。Tailwind CSSはストアフロント側で読み込み、Solidus管理画面へ意図せず適用されないことを確認します。Tailwind Plusのコンポーネントを使う場合は、ライセンス、Tailwindのメジャーバージョン、ビルド対象パスも確認します。
- 確認結果: Solidus BackendがSprocketsと `sassc-rails` を依存に含み、SassC compressorがTailwindのmodern color syntaxを処理できず、導入前の `assets:precompile` はExecJS runtime未検出で失敗しました。本番のCSS compressorを無効にした構成では、Node.jsをPATHから外しても本番precompileが成功することを確認しました。`tailwindcss-rails` はSprockets compressorが無効の場合、本番ビルドでTailwind自身の `--minify` を使います。このためNode.js追加は不要と判断して取り除きました。Sprockets経由のTailwind以外のCSSは圧縮されないため、成果物サイズは引き続き確認します。
- 注意: Starter Frontendのv4.7テンプレートは `tailwindcss-rails ~> 3.0` を指定しています。「Solidus 4.3のTailwindテーマ」はTailwind CSS 4.3を意味しません。

### S6. テストフレームワーク

- 状態: Minitest採用済み。Starter FrontendのRSpec資産の扱いは未決定
- 推奨: 既存の `docs/decisions/0001-use-minitest.md` に従い、ホストアプリではRSpecとMinitestを混在させません。Starter FrontendのRSpecテストは仕様の参考にし、重要なストアフロントの振る舞いだけをMinitestで検証します。すべての生成RSpecテストを一対一で移植することは前提にしません。
- 必須テスト: 主要な商品閲覧、カート、チェックアウト、認証から注文所有者への紐付け、ポイント利用をユーザー視点で検証します。

### S7. Solidus管理画面の拡張

- 状態: 未決定。管理機能を追加する際に判断
- 推奨: 管理画面はSolidusの `/admin` と `Spree::Admin` を基本とし、顧客向け `Storefront::` と混同しません。独自機能を `/admin` に追加する場合は、Solidus管理画面の認証・認可・レイアウトに統合したうえで、既存route name/path/actionとの重複を避けます。設定・メニュー・イベント等の公開拡張点を優先し、Controller overrideが必要な場合は[Solidusカスタマイズ方針](solidus-customization.md)に従います。

### S8. 店舗・運用設定

- 状態: 初期対象地域、全国配送、税込表示、軽減税率対象商品の有無、Stripeカード決済は決定。Stripeアカウント設定・税率設定・送料等は未決定
- 決定: 初期の販売・配送対象は日本国内のみとします。海外への販売・配送は初期スコープに含めず、必要になった時点で別途判断します。
- 決定: 初期配送範囲は日本全国（離島を含む）とします。配送業者、送料、遠隔地追加料金、配送日数は別途決定します。
- 決定: 消費者向けの商品価格は税込表示を基本とします。適用税率や軽減税率対象商品は商品カテゴリと会計要件を確認して決定します。
- 決定: 軽減税率対象商品も初期販売に含みます。商品ごとに税区分を設定できる構成とし、適用率・対象商品の判定は会計担当または専門家に確認します。
- 決定: 初期リリースはStripe経由のクレジットカード決済を提供し、その他の決済手段は後続の検討とします。
- 決定: 公式の `solidus_stripe` 拡張による画面内決済を使います。Stripe.jsとPayment Intentsを利用し、3D Secure等の追加認証を含む注文状態遷移を検証します。Stripe Checkoutのホスト型ページは採用しません。
- 運用条件: 初期リリースでは決済手段をカードのみに制限します。Stripeの秘密鍵・Webhook secretは環境変数またはRails Credentialsで管理し、ソースへ含めません。
- 互換性確認: `solidus_stripe` 5.xの依存条件はSolidus Core 3.2以上5未満で、Solidus 4.7系と互換範囲です。アプリにはまだ追加していないため、導入時にbundle解決、Stripe test mode、Webhook、3D Secure、取消・返金を検証します。
- 参考: [solidus_stripe](https://github.com/solidusio-contrib/solidus_stripe)、[Stripe Payment Intents](https://docs.stripe.com/payments/payment-intents)
- 決めること: JPYと日本語ロケール、消費税の計算・表示方法、国内住所と送料、Stripeアカウント/Webhook設定、在庫管理、注文番号、メール送信、返品・返金方針
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