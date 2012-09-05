# 序曲

> スタイルとは、偉人から良い所を分離したものである。<br/>
> -- Bozhidar Batsov

このガイドのゴールは、Ruby on Rails 3開発のための1セットのベストプラクティスおよびスタイル規則を示すことです。これは、コミュニティー駆動ですでに存在する [Ruby coding style guide](https://github.com/bbatsov/ruby-style-guide) の補足的なガイドです。

ガイドの中では「[Railsアプリケーションのテスト](#testing)」は「[Railsアプリケーション開発](#developing)」の後にあります。私は、[振る舞い駆動開発 (BDD)](http://en.wikipedia.org/wiki/Behavior_Driven_Development) がソフトウェアを開発する最良の方法であると本当に信じています。それを覚えておいてください。

Railsは信念の強いフレームワークです。そしてこれは信念の強いガイドです。心の中では、[RSpec](https://www.relishapp.com/rspec)がTest::Unitより優れていると完全に確信しています。[Sass](http://sass-lang.com/)はCSSより優れています。そして、[Haml](http://haml-lang.com/) ([Slim](http://slim-lang.com/))はErbより優れています。したがって、Test::Unit、CSS、Erbに関するどんな助言も、この中で見つけることは期待できません。

ここにある助言うちのいくつかは、Rails 3.1以上でのみ適用できます。

[Transmuter](https://github.com/TechnoGate/transmuter)を使用して、PDFあるいはこのガイドのHTMLコピーを生成することができます。

# 目次

* [Railsアプリケーション開発](#developing-rails-applications)
    * [設定](#configuration)
    * [ルーティング](#routing)
    * [コントローラ](#controllers)
    * [モデル](#models)
    * [マイグレーション](#migrations)
    * [ビュー](#views)
    * [国際化](#internationalization)
    * [アセット](#assets)
    * [Mailer](#mailers)
    * [Bundler](#bundler)
    * [有用なgem](#priceless-gems)
    * [欠陥のgem](#flawed-gems)
    * [プロセス管理](#managing-processes)
* [Railsアプリケーションのテスト](#testing-rails-applications)
    * [Cucumber](#cucumber)
    * [RSpec](#rspec)

# <a name="developing-rails-applications"> Railsアプリケーション開発

## <a name="configuration"> 設定

* `config/initializers`にカスタム初期化コードを入れてください。initializersの中のコードはアプリケーション起動時に実行されます。
* 各gemの初期化コードは、gemと同じ名前の個別のファイルにあるはずです。例えば、`carrierwave.rb`、`active_admin.rb`などです。
* development、test、productionの各環境の設定 (`config/environments/`の下の対応するファイル) に従って調節します。
  * (もしあれば) プリコンパイルの追加アセットを記します。

        ```Ruby
        # config/environments/production.rb
        # Precompile additional assets (application.js, application.css, and all non-JS/CSS are already added)
        config.assets.precompile += %w( rails_admin/rails_admin.css rails_admin/rails_admin.js )
        ```

* `production`環境に似た、`staging`環境を追加で作成してください。

## <a name="routing"> ルーティング

* RESTfulリソース (本当に必要ですか？) に対して、さらにアクションを追加する必要がある場合、`member`および`collection`ルートを使用します。

    ```Ruby
    # 悪い
    get 'subscriptions/:id/unsubscribe'
    resources :subscriptions

    # 良い
    resources :subscriptions do
      get 'unsubscribe', :on => :member
    end

    # 悪い
    get 'photos/search'
    resources :photos

    # 良い
    resources :photos do
      get 'search', :on => :collection
    end
    ```

* 複数の`member/collection`ルートを定義する必要がある場合は、代替ブロック・シンタックスを使用します。

    ```Ruby
    resources :subscriptions do
      member do
        get 'unsubscribe'
        # more routes
      end
    end

    resources :photos do
      collection do
        get 'search'
        # more routes
      end
    end
    ```

* ActiveRecordモデルの関係をよりよく表現するために、入れ子のルートを使用してください。

    ```Ruby
    class Post < ActiveRecord::Base
      has_many :comments
    end

    class Comments < ActiveRecord::Base
      belongs_to :post
    end

    # routes.rb
    resources :posts do
      resources :comments
    end
    ```

* 関連するアクションをグループ化するためにnamespaceを使用してください。

    ```Ruby
    namespace :admin do
      # Directs /admin/products/* to Admin::ProductsController
      # (app/controllers/admin/products_controller.rb)
      resources :products
    end
    ```

* 古い記法のワイルド・コントローラ・ルートを使用しないでください。このルートはGETリクエストによってアクセス可能なすべてのコントローラの中ですべてのアクションを生成します。

    ```Ruby
    # とても悪い
    match ':controller(/:action(/:id(.:format)))'
    ```

## <a name="controllers"> コントローラ

* コントローラは皮だけの状態を保ってください。これらは単にビュー層のためのデータを取り出すだけのものであるべきであり、ビジネスロジックを含むべきではありません。(すべてのビジネスロジックは、当然モデルに存在するべきです)
* それぞれのコントローラー・アクションは、初期のfindとnew以外、(理想的には) 1つのメソッドだけを起動するべきです。
* コントローラーとビューの間の変数の共有は、2つを超えない範囲にしてください。

## <a name="models"> モデル

* 非ActiveRecordモデルのクラスは自由に導入してください。
* モデルには、略語のない意味のある (しかし短い) 名前を付けてください。
* ActiveRecordのバリデーションのような振る舞いをサポートするモデルオブジェクトが必要な場合は、[ActiveAttr](https://github.com/cgriego/active_attr) gemを使用してください。

    ```Ruby
    class Message
      include ActiveAttr::Model

      attribute :name
      attribute :email
      attribute :content
      attribute :priority

      attr_accessible :name, :email, :content

      validates_presence_of :name
      validates_format_of :email, :with => /^[-a-z0-9_+\.]+\@([-a-z0-9]+\.)+[a-z0-9]{2,4}$/i
      validates_length_of :content, :maximum => 500
    end
    ```

    より完全な例は [RailsCast on the subject](http://railscasts.com/episodes/326-activeattr) を参照してください。

### ActiveRecord

* 非常に十分な理由 (あなたの管理下にないデータベースを使用する場合など) がない限りは、ActiveRecordデフォルト (テーブル名、主キーなど) を変更しないようにしてください。
* マクロスタイルのメソッド (`has_many`, `validates`, など) はクラス定義の始めにまとめてください。
* `has_and_belongs_to_many`よりも`has_many :through`を好んでください。`has_many :through`を使うことは、結合モデルに対して追加の属性とバリデーションを許可します。

    ```Ruby
    # has_and_belongs_to_many を使用
    class User < ActiveRecord::Base
      has_and_belongs_to_many :groups
    end

    class Group < ActiveRecord::Base
      has_and_belongs_to_many :users
    end

    # 好ましい方法 - has_many :through を使用
    class User < ActiveRecord::Base
      has_many :memberships
      has_many :groups, through: :memberships
    end

    class Membership < ActiveRecord::Base
      belongs_to :user
      belongs_to :group
    end

    class Group < ActiveRecord::Base
      has_many :memberships
      has_many :users, through: :memberships
    end
    ```

* 常に新しい ["sexy" validations](http://thelucid.com/2010/01/08/sexy-validation-in-edge-rails-rails-3/) を使用してください。
* カスタムバリデーションが2度以上使用されるか、バリデーションが正規表現マッチングである場合は、カスタムバリデータファイルを作成してください。

    ```Ruby
    # 悪い
    class Person
      validates :email, format: { with: /^([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})$/i }
    end

    # 良い
    class EmailValidator < ActiveModel::EachValidator
      def validate_each(record, attribute, value)
        record.errors[attribute] << (options[:message] || 'is not a valid email') unless value =~ /^([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})$/i
      end
    end

    class Person
      validates :email, email: true
    end
    ```

* すべてのカスタムバリデータは共有gemに移動されるべきです。
* named scopeは自由に使用してください。
* ラムダとパラメータを使用したnamed scope定義が複雑になる場合、named scopeと同じ目的のために、代わりに`ActiveRecord::Relation`オブジェクトを返すクラスメソッドを作るのは望ましいことです。
* `update_attribute`メソッドの振る舞いに用心してください。これはモデルバリデーション を実行せず (`update_attributes`と異なる)、容易にモデルの状態を悪くするかもしれません。
* ユーザー・フレンドリーなURLを使用してください。URLの中ではモデルの`id`ではなく、モデルの記述的な属性を表示してください。このためのいくつかの方法があります。
  * `to_param`メソッドをオーバーライドしてください。これはオブジェクトへのURLを構築するためにRailsによって使用されます。デフォルト実装では、レコードの`id`を文字列として返します。他の「人間が判読可能な」属性を含めるために、これをオーバーライドすることができます。

        ```Ruby
        class Person
          def to_param
            "#{id} #{name}".parameterize
          end
        end
        ```
        URLフレンドリーな値に変換するために、文字列の`parameterize`を呼ぶ必要があります。ActiveRecordの`find`メソッドで見つけることができるように、オブジェクトの`id`が始めにある必要があります。

  * `friendly_id` gemを使用してください。これは、その`id`の代わりにモデルの記述的な属性を使用することによって、人間が判読可能なURLを生成できるようにします。

        ```Ruby
        class Person
          extend FriendlyId
          friendly_id :name, use: :slugged
        end
        ```

        使い方について、より詳細は [gemドキュメント](https://github.com/norman/friendly_id) を確認してください。

### ActiveResource

* レスポンスが既存のもの (XMLとJSON) と異なるフォーマットであるか、これらのフォーマットの付加的な解析が必要な場合、自分のカスタムフォーマットを作成して、クラスの中でそれを使用してください。カスタム形式は次の4つのメソッドを実装するべきです：`extension`, `mime_type`, `encode`, `decode`

    ```Ruby
    module ActiveResource
      module Formats
        module Extend
          module CSVFormat
            extend self

            def extension
              'csv'
            end

            def mime_type
              'text/csv'
            end

            def encode(hash, options = nil)
              # Encode the data in the new format and return it
            end

            def decode(csv)
              # Decode the data from the new format and return it
            end
          end
        end
      end
    end

    class User < ActiveResource::Base
      self.format = ActiveResource::Formats::Extend::CSVFormat

      ...
    end
    ```

* リクエストが拡張子なしで送られるべきである場合は、`ActiveResource::Base`の`element_path`と`collection_path`メソッドをオーバーライドし、拡張子部分を削除してください。

    ```Ruby
    class User < ActiveResource::Base
      ...

      def self.collection_path(prefix_options = {}, query_options = nil)
        prefix_options, query_options = split_options(prefix_options) if query_options.nil?
        "#{prefix(prefix_options)}#{collection_name}#{query_string(query_options)}"
      end

      def self.element_path(id, prefix_options = {}, query_options = nil)
        prefix_options, query_options = split_options(prefix_options) if query_options.nil?
        "#{prefix(prefix_options)}#{collection_name}/#{URI.parser.escape id.to_s}#{query_string(query_options)}"
      end
    end
    ```

    さらにURLの他の修正が必要な場合、これらのメソッドをオーバーライドすることができます。


## <a name="migrations"> マイグレーション

* `schema.rb`はバージョン管理下に置いてください。
* 空のデータベースを初期化するために、`rake db:migrate`の代わりに`rake db:schema:load`を使用してください。
* テストデータベースのスキーマをアップデートするには`rake db:test:prepare`を使用してください。
* テーブル自体にデフォルトを設定しないようにしてください。モデル層を代わりに使用してください。

    ```Ruby
    def amount
      self[:amount] or 0
    end
    ```

    `self[:attr_name]`の使用はかなり慣用的だと思いますが、代わりにやや冗長な (そしておそらく、より読みやすい) `read_attribute`の使用を検討してもいいかもしれません：

    ```Ruby
    def amount
      read_attribute(:amount) or 0
    end
    ```

* 構造変更的なマイグレーションを書くとき (テーブルやカラムの追加など) の、新しいRails 3.1における方法は、 - `up`、`down`メソッドの代わりに、`change`メソッドを使用することです。

    ```Ruby
    # 古い方法
    class AddNameToPerson < ActiveRecord::Migration
      def up
        add_column :persons, :name, :string
      end

      def down
        remove_column :person, :name
      end
    end

    # 好ましい新しい方法
    class AddNameToPerson < ActiveRecord::Migration
      def change
        add_column :persons, :name, :string
      end
    end
    ```

## <a name="views"> ビュー

* ビューからモデル層を直接呼び出さないでください。
* ビューの中で複雑な体裁出力を作らないでください。ビューヘルパー、またはモデルのメソッドに体裁を出してください。
* 部分テンプレートやレイアウトを使用してコードの重複を軽減させてください。
* カスタムバリデータに [client side validation](https://github.com/bcardarella/client_side_validations) を追加してください。このためのステップは次の通りです。
  * `ClientSideValidations::Middleware::Base`を拡張するカスタムバリデータを宣言します。

        ```Ruby
        module ClientSideValidations::Middleware
          class Email < Base
            def response
              if request.params[:email] =~ /^([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})$/i
                self.status = 200
              else
                self.status = 404
              end
              super
            end
          end
        end
        ```

  * 新しいファイル`public/javascripts/rails.validations.custom.js.coffee`を作り、`application.js.coffee`ファイルに参照を追加します。

        ```Ruby
        # app/assets/javascripts/application.js.coffee
        #= require rails.validations.custom
        ```

  * クライアントサイドバリデータを追加します。

        ```Ruby
        #public/javascripts/rails.validations.custom.js.coffee
        clientSideValidations.validators.remote['email'] = (element, options) ->
          if $.ajax({
            url: '/validators/email.json',
            data: { email: element.val() },
            async: false
          }).status == 404
            return options.message || 'invalid e-mail format'
        ```

## <a name="internationalization"> 国際化

* 文字列や他のロケール固有の設定は、ビュー、モデル、コントローラで使用するべきではありません。これらのテキストは`config/locales`ディレクトリ内のロケールファイルに移動するべきです。
* ActiveRecordモデルのラベルを翻訳する必要がある場合は、`activerecord`スコープを使用してください：

    ```
    en:
      activerecord:
        models:
          user: Member
        attributes:
          user:
            name: "Full name"
    ```

    `User.model_name.human`は "Member" を返し、`User.human_attribute_name("name")`は "Full name" を返します。これらの属性の翻訳は、ビューのラベルとして使用されます。

* ActiveRecord属性の翻訳からのビューで使用されるテキストを分離してください。`models`フォルダの中にはモデル、`views`フォルダの中にはビューで使用されるテキスト用のローカルファイルを置いてください。
  * 追加ディレクトリからのローカルファイルの編成が完了したということは、これらのディレクトリはロードされるために`application.rb`ファイルで定義されているはずです。

        ```Ruby
        # config/application.rb
        config.i18n.load_path += Dir[Rails.root.join('config', 'locales', '**', '*.{rb,yml}')]
        ```

* `locales`ディレクトリのルート下のファイルに、日付や通貨の書式のような共有される地域化オプションを置いてください。
* I18nメソッドの短縮形を使用してください。`I18n.translate`の代わりに`I18n.t`。`I18n.localize`の代わりに`I18n.l`です。
* ビューで使われるテキストに "lazy" ルックアップを使用してください。次のような構造です。

    ```
    en:
      users:
        show:
          title: "User details page"
    ```

    `users.show.title`の値は、`app/views/users/show.html.haml`テンプレートで次のように得ることができます。

    ```Ruby
    = t '.title'
    ```

* `:scope`オプションを指定する代わりに、コントローラとモデルをドットで区切ったキーを使用してください。ドットで区切った呼び出しは、読みやすく、階層のトレースがより簡単です。

    ```Ruby
    # 方法
    I18n.t 'activerecord.errors.messages.record_invalid'

    # 代わりの方法
    I18n.t :record_invalid, :scope => [:activerecord, :errors, :messages]
    ```

* Rails I18nのより詳細な情報は以下で見つけることができます。[Rails Guides](http://guides.rubyonrails.org/i18n.html)

## <a name="assets"> アセット

アプリケーション内の構成にてこ入れするために[アセットパイプライン](http://guides.rubyonrails.org/asset_pipeline.html)を使用してください。

* カスタムのstyleshees, javascripts, imagesのために`app/assets`を使用してください。
* [jQuery](http://jquery.com/) や [bootstrap](http://twitter.github.com/bootstrap/) のような第三者のコードは`vendor/assets`に置かれるべきです。
* 可能な場合は、アセットをgem化したバージョンを使用してください。(例：[jquery-rails](https://github.com/rails/jquery-rails))

## <a name="mailers"> Mailer

* Mailerの名前は`SomethingMailer`としてください。接尾辞のないMailerでは、どれがMailerか、どのビューがMailerと関係があるか、明白ではありません。
* HTMLとプレーンテキストの両方のビューテンプレートを提供してください。
* development環境中でメール配信に失敗した際のエラー発生を有効にしてください。エラーはデフォルトでは無効です。

    ```Ruby
    # config/environments/development.rb

    config.action_mailer.raise_delivery_errors = true
    ```

* development環境ではSMTPサーバに`smtp.gmail.com`を使用してください。(もちろん、あなたがローカルSMTPサーバを持っていない場合です)

    ```Ruby
    # config/environments/development.rb

    config.action_mailer.smtp_settings = {
      address: 'smtp.gmail.com',
      # more settings
    }
    ```

* ホスト名にデフォルト設定を与えてください。

    ```Ruby
    # config/environments/development.rb
    config.action_mailer.default_url_options = {host: "#{local_ip}:3000"}

    # config/environments/production.rb
    config.action_mailer.default_url_options = {host: 'your_site.com'}

    # in your mailer class
    default_url_options[:host] = 'your_site.com'
    ```

* メールの中でサイトへのリンクを使用する必要がある場合は、`_path`メソッドではなく、常に`_url`を使用してください。`_url`メソッドはホスト名を含んでいますが、`_path`メソッドは含んでいません。

    ```Ruby
    # 間違い
    You can always find more info about this course
    = link_to 'here', url_for(course_path(@course))

    # 正しい
    You can always find more info about this course
    = link_to 'here', url_for(course_url(@course))
    ```

* FromとToのアドレスに適切な書式を使ってください。次のような書式を使ってください。

    ```Ruby
    # in your mailer class
    default from: 'Your Name <info@your_site.com>'
    ```

* test環境のためのメールdelivery methodには`test`が設定されていることを確認してください。

    ```Ruby
    # config/environments/test.rb

    config.action_mailer.delivery_method = :test
    ```

* developmentとproduction環境のためのメールdelivery methodには`smtp`が設定されていることを確認してください。

    ```Ruby
    # config/environments/development.rb, config/environments/production.rb

    config.action_mailer.delivery_method = :smtp
    ```

* いくつかのメールクライアントが外部スタイルの問題を抱えているため、HTMLメールを送るときは、すべてのスタイルがインラインでなければなりません。しかしながら、これはメンテンナンスを困難にし、コードの重複にもつながります。スタイルを変換し、それを対応するHTMLタグに挿入するための、2つの同じようなgemがあります。[premailer-rails3](https://github.com/fphilipe/premailer-rails3) と [roadie](https://github.com/Mange/roadie) です。
* ページレスポンスを生成する間にメールを送ることは避けるべきです。ページの読み込みに遅延が発生しますし、もし複数のメールが送信されるのなら、リクエストがタイムアウトする場合があります。これを克服するためには、[delayed_job](https://github.com/tobi/delayed_job) gemの助けを借りてバックグラウンドプロセスで送信します。

## Bundler

* 開発またはテストのみ使用されるgemは、Gemfileの適切なグループの中に置いてください。
* プロジェクトでは定評のあるgemだけ使用してください。ほとんど知られていないgemを含めることを検討しているのなら、そのソースコードの注意深い調査を最初に行うべきです。
* OS特有のgemは、異なるOSを使用している複数の開発者と一緒のプロジェクトのために、頻繁に変わる`Gemfile.lock`によって得ます。OS Xの特有のgemをすべてGemfileの中の`darwin`グループに、Linux特有のgemを`linux`グループに加えてください。

    ```Ruby
    # Gemfile
    group :darwin do
      gem 'rb-fsevent'
      gem 'growl'
    end

    group :linux do
      gem 'rb-inotify'
    end
    ```

    正しい環境で適切なgemを要求するためには、`config/application.rb`に以下を加えてください。

    ```Ruby
    platform = RUBY_PLATFORM.match(/(linux|darwin)/)[0].to_sym
    Bundler.require(platform)
    ```

* バージョン管理から`Gemfile.lock`を取り除かないでください。これは任意に生成されたファイルではありません。- これは`bundle install`した時に、チームメンバーが確実にすべて同じgemバージョンを得るためのものです。

## <a name="priceless-gems"> 有用なgem

最も重要なプログラミングの法則の1つはこうです。「車輪を再発明するな！」。あるタスクに直面した時には、自分のものを広げる前に、常に既存の解決策を見つけるために周りを見回すべきです。ここにあるリストは、多くのRailsプロジェクトに役立つ「有用な」gemです。(すべてRails 3.1準拠)

* [active_admin](https://github.com/gregbell/active_admin) - ActiveAdminがあれば、Railsアプリにおける管理者インタフェースの作成は子供の遊びのようなものです。素敵なダッシュボード、CRUD UI、その他いろいろ。とても柔軟でカスタマイズ可能です。
* [capybara](https://github.com/jnicklas/capybara) - Capybaraは、Rails、Sinatra、MerbのようなRackアプリケーションにおける統合テストのプロセスを単純化することを目標としています。CapybaraはリアルユーザーのWebアプリケーションとの対話をシミュレートします。それはテストを実行するドライバーに関して不可知論者で、Rack::Testに付属し、Seleniumをビルトインでサポートしています。HtmlUnit、WebKit、env.jsは外部gemによってサポートされます。RSpecとCucmberとのコンビネーションで素晴らしい仕事をします。
* [carrierwave](https://github.com/jnicklas/carrierwave) - Railsにおける究極のファイルアップロードソリューション。アップロードファイルはローカルと、クラウドストレージの両方をサポートします。画像の後処理のためにImageMagickと素晴らしく統合します。
* [client_side_validations](https://github.com/bcardarella/client_side_validations) - サーバーサイドの既存のモデルバリデーションから、自動的にJavaScriptクライアントサイドバリデーションを生成する、素晴らしいgemです。非常にお勧めします！
* [compass-rails](https://github.com/chriseppstein/compass) - いくつかのCSSフレームワークのサポートを追加する、素晴らしいgem。CSSファイルのコードが削減され、ブラウザ非互換性との戦いを助けるsaas mixinsのコレクションが含まれています。
* [cucumber-rails](https://github.com/cucumber/cucumber-rails) - Cucumber はRubyで機能テストを開発するプレミアムツールです。cucumber-railsは、RailsへのCucumberの統合を提供します。
* [devise](https://github.com/plataformatec/devise) - Deviseは、Railsアプリケーションのフル機能の認証ソリューションです。 カスタム認証ソリューションを展開するほとんどの場合、Deviseの使用が適しています。
* [fabrication](http://fabricationgem.org/) - 素晴らしいfixtureの代替 (editor's choice).
* [factory_girl](https://github.com/thoughtbot/factory_girl) - fabricationの代わり。とても成熟したfixtureの代替です。fabricationの精神を持っています。
* [faker](http://faker.rubyforge.org/) - ダミーデータを生成する手軽なgem。(氏名、住所、その他)
* [feedzirra](https://github.com/pauldix/feedzirra) - とても高速で柔軟なRSS/Atomフィードのパーサー。
* [friendly_id](https://github.com/norman/friendly_id) - モデルのIDの代わりに記述的な属性を使って、人間が読めるURLを生成します
* [guard](https://github.com/guard/guard) - ファイルの変更を監視し、それに基づいたタスクを起動する、素晴らしいgemです。多くの有用な拡張が載せられました。自動テストとwatchrよりはるかに優れています。
* [haml-rails](https://github.com/indirect/haml-rails) - haml-railsは、HamlのRailsへの統合を提供します。
* [haml](http://haml-lang.com) - HAMLは簡潔なテンプレート言語で、多くの人に (あなたも含まれます) Erbよりはるかに優れいていると考えられています。
* [kaminari](https://github.com/amatsuda/kaminari) - 素晴らしいページングソリューション。
* [machinist](https://github.com/notahat/machinist) - fixtureは楽しくありません。mechanistなら。
* [rspec-rails](https://github.com/rspec/rspec-rails) - RSpecはTest::MiniTestの代替です。私はRSpecを十分に強く推奨することができません。rspec-railsは、RSpecのRailsへの統合を供給します。
* [simple_form](https://github.com/plataformatec/simple_form) - 一度simple_form (あるいはformtastic) を使用したならば、あなたは決してRailsのデフォルト形式に関して聞かされたくありません。フォームを構築するためにマークアップに対して文句のない素晴らしいDSLを持っています。
* [simplecov-rcov](https://github.com/fguillen/simplecov-rcov) - SimpleCovのためのRCovフォーマッタ。Hudsonのcontininousな統合サーバーとSimpleCovを使用しようとしているなら有用です。
* [simplecov](https://github.com/colszowka/simplecov) - コードカバレッジツール。RCovと異なり、Ruby 1.9と完全に互換性をもちます。素晴らしいレポートを生成します。必携!
* [slim](http://slim-lang.com) - Slimは簡潔なテンプレート言語です。HAMLより優れています (Erbには言及しない)。私が使用するのを止める重くて細いただ1つの理由は、主要なエディタ/IDEのサポートの不足です。そのパフォーマンスは驚異的です。
* [spork](https://github.com/sporkrb/spork) - フレームワーク (現状、RSpec/Cucumber) をテストするためのDRbサーバです。クリーンなテスト状態を保障するために各々の実行前にフォークします。簡単に言えば、多くのテスト環境を事前ロードします。その結果として、テストの最初の時間が大幅に減少します。絶対に必携。
* [sunspot](https://github.com/sunspot/sunspot) - SOLRによる全文検索エンジン。

このリストは完全ではありません。また、他のgemは今後加えられるかもしれません。リストのgemはすべてフィールドテストされています。これらすべては活発な開発およびコミュニティ活動をしており、よいコード品質であることが知られています。

## <a name="flawed-gems">欠陥のgem

これは問題があるか、他のgemによって取って代わられるgemのリストです。プロジェクトの中でこれらを使用しないようにするべきです。

* [rmagick](http://rmagick.rubyforge.org/) - このgemはメモリ消費で悪名高い。代わりに [minimagick](https://github.com/probablycorey/mini_magick) を使用してください。
* [autotest](http://www.zenspider.com/ZSS/Products/ZenTest/) - テストを自動的に実行するための古いソリューションです。guard と [watchr](https://github.com/mynyml/watchr) よりも遥かに劣っています。
* [rcov](https://github.com/relevance/rcov) - コードカバレッジツールです。Ruby 1.9と互換性がありません。[SimpleCov](https://github.com/colszowka/simplecov) を使用してください。
* [therubyracer](https://github.com/cowboyd/therubyracer) - 非常に大量のメモリを使用するので、本番環境でのこのgemの使用には強く反対します。[Mustang](https://github.com/nu7hatch/mustang) の利用を提案します。

このリストは作成途中です。他にもポピュラーであるが欠陥のあるgemを知っていれば教えてください。

## <a name="managing-processes"> プロセス管理

* プロジェクトが様々な外部プロセスに依存している場合は、それらを管理するために [foreman](https://github.com/ddollar/foreman) を使います。

# <a name="testing-rails-applications"> Railsアプリケーションのテスト

新しい機能を実装するためのの最良のアプローチは、おそらくBDDアプローチです。いくつかのハイレベルの機能テスト (一般にCucumberを使用して書かれる) を書くことにより開始します。そして、機能の実装を洗い出すためにこれらのテストを使用します。始めに機能用のビューのspecを書き、適切なビューを作成するためにそれらのspecを使用します。後でビューにデータを与えて、コントローラを実装するために、それらのspecを使用するコントローラ用のspecを作成します。最後に、モデルspecおよびモデル自身を実装します。

## <a name="cucumber"> Cucumber

* 保留中のシナリオに`@wip` (work in progress: 作業中) タグを付けてください。これらのシナリオは無視され、失敗としてマークされません。保留中のシナリオの作業が完了し、テストするための機能性が実装されたら、テストスイートにこのシナリオを含めるために`@wip`タグを削除します。
* `@javascript`タグが付いたシナリオを除外するようにデフォルトのプロフィールをセットアップしてください。それらはテストにブラウザを使用します。通常のシナリオの実行速度を上げるために、それらを無効にすることをお勧めします。
* `@javascript`タグが付けられたシナリオのための別のプロフィールをセットアップしてください。
  * プロフィールは`cucumber.yml`ファイルで設定することができます。

        ```Ruby
        # definition of a profile:
        profile_name: --tags @tag_name
        ```

  * プロフィールは次のコマンドで実行されます。

        ```
        cucumber -p profile_name
        ```

* fixtureの代わりに [fabrication](http://fabricationgem.org/) 使用している場合は、あらかじめ定義された [fabrication steps](http://fabricationgem.org/#!cucumber-steps) を使用してください。
* 古い`web_steps.rb`ステップ定義を使わないでください！[WebステップはCucumberの最新バージョンで削除されています。](http://aslakhellesoy.com/post/11055981222/the-training-wheels-came-off)その使い方は、適切にアプリケーションのドメインを反映しない冗長なシナリオの作成につながります。
* 要素idではなく、可視のテキスト (リンク、ボタンなど) で要素の存在を調べる場合。これは、i18nに関する問題を見つけることができます。
* 同じ種類のオブジェクトの異なる機能のために、個別のフィーチャを作ってください：

    ```Ruby
    # 悪い
    Feature: Articles
    # ... フィーチャ実装 ...

    # 良い
    Feature: Article Editing
    # ... フィーチャ実装 ...

    Feature: Article Publishing
    # ... フィーチャ実装 ...

    Feature: Article Search
    # ... フィーチャ実装 ...

    ```

* 各々のフィーチャには3つの主成分があります。
  * タイトル
  * ナラティブ (物語) - フィーチャについての短い説明。
  * 合格基準 - 個々のステップから構成されたシナリオのセット。
* 最も一般的なフォーマットがConnextraフォーマットとして知られています。

    ```Ruby
    In order to [benefit] ...
    A [stakeholder]...
    Wants to [feature] ...
    ```

このフォーマットは最も一般的なものですが、必須ではありません、ナラティブはフィーチャの複雑さに応じて自由なテキストになりえます。

* シナリオのDRYを維持するために、シナリオアウトラインを自由に使用してください。

    ```Ruby
    Scenario Outline: User cannot register with invalid e-mail
      When I try to register with an email "<email>"
      Then I should see the error message "<error>"

    Examples:
      |email         |error                 |
      |              |The e-mail is required|
      |invalid email |is not a valid e-mail |
    ```

* シナリオ用のステップは`step_definitions`ディレクトリの下の`.rb`ファイルにあります。ステップファイルのファイル名命名規則は`[description]_steps.rb`です。ステップは異なる基準に基づいた異なるファイルへ分けることができます。各フィーチャ (`home_page_steps.rb`) のために1ステップのファイルを持つことは可能です。また、特定のオブジェクト(`articles_steps.rb`) のためにすべてのフィーチャの1ステップのファイルがさらにある場合もあります。
* 繰り返しを回避するために複数行ステップ引数を使用してください。

    ```Ruby
    Scenario: User profile
      Given I am logged in as a user "John Doe" with an e-mail "user@test.com"
      When I go to my profile
      Then I should see the following information:
        |First name|John         |
        |Last name |Doe          |
        |E-mail    |user@test.com|

    # the step:
    Then /^I should see the following information:$/ do |table|
      table.raw.each do |field, value|
        find_field(field).value.should =~ /#{value}/
      end
    end
    ```

* シナリオのDRYを保つために、複合ステップを使用してください。

    ```Ruby
    # ...
    When I subscribe for news from the category "Technical News"
    # ...

    # the step:
    When /^I subscribe for news from the category "([^"]*)"$/ do |category|
      steps %Q{
        When I go to the news categories page
        And I select the category #{category}
        And I click the button "Subscribe for this category"
        And I confirm the subscription
      }
    end
    ```
* 常にshould_not肯定の代わりにCapybaraの否定マッチャーを使用してください。Ajaxアクションをテストすることを可能にし、指定されたタイムアウトでマッチングの再試行を行います。[さらに多くの説明は、CapybaraのREADMEを参照してください。](https://github.com/jnicklas/capybara)

## <a name="rspec"> RSpec

* exampleごとに1つの期待値だけを使用してください。

    ```Ruby
    # 悪い
    describe ArticlesController do
      #...

      describe 'GET new' do
        it 'assigns new article and renders the new article template' do
          get :new
          assigns[:article].should be_a_new Article
          response.should render_template :new
        end
      end

      # ...
    end

    # 良い
    describe ArticlesController do
      #...

      describe 'GET new' do
        it 'assigns a new article' do
          get :new
          assigns[:article].should be_a_new Article
        end

        it 'renders the new article template' do
          get :new
          response.should render_template :new
        end
      end

    end
    ```

* `describe`と`context`を多用してください。
* `describe`ブロックの命名は次のようにしてください。
  * メソッド以外は "description" とする。
  * メソッドには "#method" 「#」を使う。
  * クラスメソッドには ".method" 「.」を使う。

    ```Ruby
    class Article
      def summary
        #...
      end

      def self.latest
        #...
      end
    end

    # the spec...
    describe Article
      describe '#summary'
        #...
      end

      describe '.latest'
        #...
      end
    end
    ```

* テストオブジェクトの生成には [fabricators](http://fabricationgem.org/) を使用してください。
* モックとスタブを多用してください。

    ```Ruby
    # モデルをモック
    article = mock_model(Article)

    # メソッドをスタブ
    Article.stub(:find).with(article.id).and_return(article)
    ```

* モデルをモックする場合、`as_null_object`メソッドを使用してください。これは、期待するメッセージにのみをリスニングし、他のメッセージを無視するように出力に命じます。

    ```Ruby
    article = mock_model(Article).as_null_object
    ```

* exampleのためにデータを作る際は、`before(:each)`ブロックの代わりに`let`ブロックを使用してください。

    ```Ruby
    # こうしてください
    let(:article) { Fabricate(:article) }

    # ... これよりも
    before(:each) { @article = Fabricate(:article) }
    ```

* できれば`subject`を使用してください。

    ```Ruby
    describe Article do
      subject { Fabricate(:article) }

      it 'is not published on creation' do
        subject.should_not be_published
      end
    end
    ```

* できれば`specify`を使用してください。これは`it`の同意語ですが、docstringがない場合、より読みやすいです。

    ```Ruby
    # 悪い
    describe Article do
      before { @article = Fabricate(:article) }

      it 'is not published on creation' do
        @article.should_not be_published
      end
    end

    # 良い
    describe Article do
      let(:article) { Fabricate(:article) }
      specify { article.should_not be_published }
    end
    ```

* できれば`its`を使用してください。

    ```Ruby
    # 悪い
    describe Article do
      subject { Fabricate(:article) }

      it 'has the current date as creation date' do
        subject.creation_date.should == Date.today
      end
    end

    # 良い
    describe Article do
      subject { Fabricate(:article) }
      its(:creation_date) { should == Date.today }
    end
    ```

### ビュー

* ビューのspec`spec/views`のディレクトリ構造は、`app/views`の中のものと一致します。
例えば、`app/views/users`の中のspecは`spec/views/users`に置かれます。
* ビューのspecの命名規則はビューの名前に`_spec.rb`を視界名に加えたものです。例えば、ビュー`_form.html.haml`には対応するspec`_form.html.haml_spec.rb`があります。
* `spec_helper.rb`は各ビューspecファイルの中で要求されるために必要です。
* 外側の`describe`ブロックには、`app/views`なしのビューへのパスを使用します。引数なしで呼ばれた場合、`render`メソッドによって使用されます。

    ```Ruby
    # spec/views/articles/new.html.haml_spec.rb
    require 'spec_helper'

    describe 'articles/new.html.haml' do
      # ...
    end
    ```

* ビューspec中では常にモデルをモックしてください。ビューの目的は情報を単に表示することだけです。
* `assign`メソッドは、コントローラーによって供給されビューが使用する変数を供給します。

    ```Ruby
    # spec/views/articles/edit.html.haml_spec.rb
    describe 'articles/edit.html.haml' do
    it 'renders the form for a new article creation' do
      assign(
        :article,
        mock_model(Article).as_new_record.as_null_object
      )
      render
      rendered.should have_selector('form',
        method: 'post',
        action: articles_path
      ) do |form|
        form.should have_selector('input', type: 'submit')
      end
    end
    ```

* should_not肯定よりもCapybaraの否定セレクタを使ってください。

    ```Ruby
    # 悪い
    page.should_not have_selector('input', type: 'submit')
    page.should_not have_xpath('tr')

    # 良い
    page.should have_no_selector('input', type: 'submit')
    page.should have_no_xpath('tr')
    ```

* ビューがヘルパーメソッドを使用する場合、これらのメソッドをスタブする必要があります。ヘルパーメソッドのスタブは`template`オブジェクト上で行われます。

    ```Ruby
    # app/helpers/articles_helper.rb
    class ArticlesHelper
      def formatted_date(date)
        # ...
      end
    end

    # app/views/articles/show.html.haml
    = "Published at: #{formatted_date(@article.published_at)}"

    # spec/views/articles/show.html.haml_spec.rb
    describe 'articles/show.html.haml' do
      it 'displays the formatted date of article publishing'
        article = mock_model(Article, published_at: Date.new(2012, 01, 01))
        assign(:article, article)

        template.stub(:formatted_date).with(article.published_at).and_return('01.01.2012')

        render
        rendered.should have_content('Published at: 01.01.2012')
      end
    end
    ```

* ヘルパーのspecはビューspecから分けられ、`spec/helpers`ディレクトリに置かれます。

### コントローラ

* モデルをモックしてメソッドをスタブしてください。コントローラーのテストはモデル生成に左右されるべきではありません。
* コントローラーが責任を負うべき振る舞いだけをテストしてください：
  * 特別なメソッドを実行してください。
  * アクションから返されたデータ - assigns、など
  * アクションからの結果 - template、render、redirect、など

        ```Ruby
        # Example of a commonly used controller spec
        # spec/controllers/articles_controller_spec.rb
        # We are interested only in the actions the controller should perform
        # So we are mocking the model creation and stubbing its methods
        # And we concentrate only on the things the controller should do

        describe ArticlesController do
          # The model will be used in the specs for all methods of the controller
          let(:article) { mock_model(Article) }

          describe 'POST create' do
            before { Article.stub(:new).and_return(article) }

            it 'creates a new article with the given attributes' do
              Article.should_receive(:new).with(title: 'The New Article Title').and_return(article)
              post :create, message: { title: 'The New Article Title' }
            end

            it 'saves the article' do
              article.should_receive(:save)
              post :create
            end

            it 'redirects to the Articles index' do
              article.stub(:save)
              post :create
              response.should redirect_to(action: 'index')
            end
          end
        end
        ```

* アクションが受信パラメータに応じて異なる振る舞いをする場合、contextを使用してください。

    ```Ruby
    # A classic example for use of contexts in a controller spec is creation or update when the object saves successfully or not.

    describe ArticlesController do
      let(:article) { mock_model(Article) }

      describe 'POST create' do
        before { Article.stub(:new).and_return(article) }

        it 'creates a new article with the given attributes' do
          Article.should_receive(:new).with(title: 'The New Article Title').and_return(article)
          post :create, article: { title: 'The New Article Title' }
        end

        it 'saves the article' do
          article.should_receive(:save)
          post :create
        end

        context 'when the article saves successfully' do
          before { article.stub(:save).and_return(true) }

          it 'sets a flash[:notice] message' do
            post :create
            flash[:notice].should eq('The article was saved successfully.')
          end

          it 'redirects to the Articles index' do
            post :create
            response.should redirect_to(action: 'index')
          end
        end

        context 'when the article fails to save' do
          before { article.stub(:save).and_return(false) }

          it 'assigns @article' do
            post :create
            assigns[:article].should be_eql(article)
          end

          it 're-renders the "new" template' do
            post :create
            response.should render_template('new')
          end
        end
      end
    end
    ```

### モデル

* specではモデルをモックしないでください。
* 本物のオブジェクトを作るためにfabricationを使用してください。
* 他のモデルや子オブジェクトをモックすることは容認できます。
* 重複を避けるために、specにすべてのexampleのためのモデルを作成してください。

    ```Ruby
    describe Article
      let(:article) { Fabricate(:article) }
    end
    ```

* 作ったモデルが有効であることを保証するexampleを加えてください。

    ```Ruby
    describe Article
      it 'is valid with valid attributes' do
        article.should be_valid
      end
    end
    ```

* バリデーションをテストする場合、検証する必要がある属性を指定するには`have(x).errors_on`を使用してください。`be_valid`の使用は、問題が意図した属性にあることを保証するものではありません。

    ```Ruby
    # 悪い
    describe '#title'
      it 'is required' do
        article.title = nil
        article.should_not be_valid
      end
    end

    # 好ましい
    describe '#title'
      it 'is required' do
        article.title = nil
        article.should have(1).error_on(:title)
      end
    end
    ```

* バリデーションされる各属性ごとに個別の`describe`を追加してください。

    ```Ruby
    describe Article
      describe '#title'
        it 'is required' do
          article.title = nil
          article.should have(1).error_on(:title)
        end
      end
    end
    ```

* モデル属性の一意性をテストするときは、他方のオブジェクトに`another_object`という名前を付けてください。

    ```Ruby
    describe Article
      describe '#title'
        it 'is unique' do
          another_article = Fabricate.build(:article, title: article.title)
          article.should have(1).error_on(:title)
        end
      end
    end
    ```

### Mailer

Mailer specの中のモデルはモックされるべきです。Mailerはモデル生成に依存するべきではありません。
* Mailer specは以下のことを確認する必要があります：
  * 見出しが正しい。
  * 受取人のE-mailアドレスが正しい。
  * メールが正しいE-mailアドレスに送信された。
  * メールには必要な情報が含まれている。

     ```Ruby
     describe SubscriberMailer
       let(:subscriber) { mock_model(Subscription, email: 'johndoe@test.com', name: 'John Doe') }

       describe 'successful registration email'
         subject { SubscriptionMailer.successful_registration_email(subscriber) }

         its(:subject) { should == 'Successful Registration!' }
         its(:from) { should == ['info@your_site.com'] }
         its(:to) { should == [subscriber.email] }

         it 'contains the subscriber name' do
           subject.body.encoded.should match(subscriber.name)
         end
       end
     end
     ```

### アップローダー

* アップローダーに関してテストすることができるものは、画像が正しくリサイズされるかどうかです。ここでは [carrierwave](https://github.com/jnicklas/carrierwave) 画像アップローダーのサンプルを示します。

    ```Ruby

    # rspec/uploaders/person_avatar_uploader_spec.rb
    require 'spec_helper'
    require 'carrierwave/test/matchers'

    describe PersonAvatarUploader do
      include CarrierWave::Test::Matchers

      # Enable images processing before executing the examples
      before(:all) do
        UserAvatarUploader.enable_processing = true
      end

      # Create a new uploader. The model is mocked as the uploading and resizing images does not depend on the model creation.
      before(:each) do
        @uploader = PersonAvatarUploader.new(mock_model(Person).as_null_object)
        @uploader.store!(File.open(path_to_file))
      end

      # Disable images processing after executing the examples
      after(:all) do
        UserAvatarUploader.enable_processing = false
      end

      # Testing whether image is no larger than given dimensions
      context 'the default version' do
        it 'scales down an image to be no larger than 256 by 256 pixels' do
          @uploader.should be_no_larger_than(256, 256)
        end
      end

      # Testing whether image has the exact dimensions
      context 'the thumb version' do
        it 'scales down an image to be exactly 64 by 64 pixels' do
          @uploader.thumb.should have_dimensions(64, 64)
        end
      end
    end

    ```

# 参考文献

Railsのスタイルには優れたリソースがあります。時間があるなら検討してみてください。

* [The Rails 3 Way](http://tr3w.com/)
* [Ruby on Rails Guides](http://guides.rubyonrails.org/)
* [The RSpec Book](http://pragprog.com/book/achbd/the-rspec-book)

# 貢献

このガイドの中で書かれた何も石の中でセットされていません。これはRailsコーディングスタイルに興味を持っている皆と一緒に働きたいという私の願望です。だから私たちはRubyコミュニティ全体に有益となるリソースを作成することができました。

改善のためにチケットやPull Requestを自由に送ってください。あなたの助力に感謝します！

# Spread the Word

コミュニティー駆動のスタイルガイドは、その存在を知らないコミュニティーにほとんど役に立ちません。ガイドに関してツイートして、友達や同僚と共有してください。私たちが得るすべてのコメントや提案、意見はガイドをわずかにより良くします。そして私たちはできるだけ最良のガイドを持ちたいですよね。
