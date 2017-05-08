# kanrinmaru
[![Build Status](https://circleci.com/gh/serverworks/kanrinmaru.png?circle-token=fca2c70b3b156036622f1439d8f706003bb75aaf)](https://circleci.com/gh/serverworks/kanrinmaru)
## 実行に必要なもの
* 必須
   * Heroku Toolbelt (https://toolbelt.heroku.com/)
   * Ruby 2.3
   * Node.js 6.10.0, npm
   * PostgreSQL 9.6
   * Redis 2
   * PhantomJS v2.1 (http://phantomjs.org/download.html)
     * テストの実行に必要
* ~~任意~~
   * ~~Googleアカウントでのログインを利用する場合に必要~~
      * ~~Google OAuth 2.0のクライアントIDとクライアントシークレット~~
   * ~~Facebookアカウントでのログインを利用する場合に必要~~
      * ~~Facebook OAuth 2.0のアプリIDとシークレットキー~~
※Google/Facebookアカウントは現在設定不要
## AWSアカウントの準備
- IAMユーザを個人用検証アカウント等で作成
- 上記で作成したIAMのKMS(暗号化キー)を東京リージョンで作成
- SWFのドメインを作成
## セットアップ
### gemのインストール
必要なgemをインストールします。
    $ bundle install --path vendor/bundle
### npmパッケージのインストール
    $ npm install
### PhantomJSのインストール
テスト実行時に利用するブラウザとしてPhantomJSをインストールします。
Mac OS XでHomebrewを利用している場合は、以下のようにしてインストールすることができます。
    $ brew install phantomjs
もしくは http://phantomjs.org/download.html よりバイナリパッケージをダウンロードして、パスの通った場所に配置します。
以下のように`phantomjs`コマンドが実行できていればOKです。
    $ phantomjs --version
    2.1.1
### 環境変数ファイルの作成
環境変数ファイルを`.env`という名前で作成します。
    $ cp sample.env .env
    $ vi .env
| グループ | 項目名 | 設定値  |
| ------------- | ------------- | ----- |
| aws-sdk関連項目 | AWS_ACCESS_KEY_ID<br>AWS_SECRET_ACCESS_KEY | 作成したIAMの情報をセット |
| SWF関連項目 | SWF_DOMAIN_UNLIMITED<br>〜<br>SWF_DOMAIN_TRIAL | SWFのドメイン名をセット(すべて同一値でOK) |
|  | WORKFLOW_AUTOSTART | trueをセット |
| KMS関連項目 | KMS_REGION | KMSを作成したリージョン (例: "ap-northeast-1") |
|  | KMS_MASTER_KEY_ID | KMSのARN |
|  | KMS_ENCRYPTION_CONTEXT | `kanrinmaru-worker`の`.env`にある`KMS_ENCRYPTION_CONTEXT`と同じ値をセット<br>(workerのREADMEにある値でOK) |
環境変数ファイルの`AWS_ACCESS_KEY_ID`と`AWS_SECRET_ACCESS_KEY`には、システムが利用するAWSアカウントの情報を設定します（ユーザーが利用するAWSアカウントではありません）。
`GOOGLE_*`と`FACEBOOK_*`は未設定のままで構いません。ただし、テストを実行するだけの場合は `AWS_ACCESS_KEY_ID` と `AWS_SECRET_ACCESS_KEY` も設定する必要はありません。
環境変数ファイルはアプリケーションが起動する時に自動的に読み込まれます。（環境変数を変更した場合は再起動が必要）
### データベースの設定
事前にPostgreSQLをインストールし、起動しておきます。  
（Postgres.app を利用すると、ローカルで異なるPostgreSQLのメジャーバーションを併用しやすい）
開発用およびテスト用のデータベースを初期化します。
    $ bundle exec rake db:setup
    (中略)
    -- initialize_schema_migrations_table()
       -> 0.0125s
`bundle exec`を付けずに`rake`コマンドを実行すると、Railsアプリケーション起動を高速化するための仕組みによってデータベースの作成前にアプリケーションのpreloadが行われますが、その際にまだデータベースが存在していないことによってエラーが発生するので注意してください。
参考:
- http://stackoverflow.com/a/28452078
### テストの実行
セットアップが完了したことを確認するため、テストを実行します。
    $ rake spec
    (中略
    Finished in 6.99 seconds
    226 examples, 0 failures, 6 pending
    Randomized with seed 51619
`0 failures`と表示されれば、セットアップは成功しています。`6 pending` の表示は気にしなくて大丈夫です。
テストが上手く実行出来ない場合は、テスト用の環境変数ファイル `.env.test` のタイムアウトに関しての設定を少し長めに設定し直して実行してみてください。
    RACK_TIMEOUT=120
Railsアプリのロード時間を短縮してテストの実行を高速化するためにspringを利用することもできます。
    $ bundle exec bin/spring rspec spec
springの詳細な利用方法については https://github.com/rails/spring を参照してください。
### Redisの起動
Redisは、メール送信などの非同期処理のストレージとして使われます。
Redisが起動していない場合、メールの送信を伴うアクションの実行時などに例外が発生します。
RedisはHomebrewで以下のようにインストールします。
    $ brew install redis
localhostの6379ポートで待ち受けるRedisを起動します。
    $ redis-server
### アプリケーションの起動
アプリケーションの起動で `タイムアウトエラー` が出る場合は環境変数ファイル `.env` のタイムアウトに関しての設定を少し長めに設定し直して実行してみてください。
    RACK_TIMEOUT=120
#### rails serverを使う場合
rails serverを使う場合は以下のようにして、アプリケーションを起動します。
    $ rails s
[http://localhost:3000](http://localhost:3000)にアクセスすると、ログインページが開きます。
この場合、メール送信などの非同期処理を実行するために、別なターミナルでワーカープロセスを実行する必要があります。
    $ bundle exec sidekiq -q default -q mailers
ワーカープロセスを実行しない限り、非同期処理の情報はRedisに保存されるだけで、メールの送信などは行われません。
#### unicornを使う場合
アプリケーションサーバーとしてunicornを利用する場合は（production環境ではunicornが利用されます）、foremanを利用してアプリケーションを起動します。
    $ heroku local
[http://localhost:5000](http://localhost:5000)にアクセスすると、ログインページが開きます。
この場合、メール送信などの非同期処理を実行するためのワーカープロセスもあわせて実行されるので、別途ワーカープロセスを実行する必要はありません。
    $ foreman start
    19:10:15 web.1    | started with pid 24080
    19:10:15 worker.1 | started with pid 24081
    19:10:17 web.1    | I, [2014-08-22T19:10:17.637684 #24080]  INFO -- : Refreshing Gem list
    19:10:23 worker.1 | 2014-08-22T10:10:23.719Z 24081 TID-ovm3cukmg INFO: Booting Sidekiq 3.2.1 with redis options {:namespace=>"sidekiq_development"}
### RailsアプリのAPIドキュメントの生成
YARDを利用して、ソースコードからHTMLドキュメントを生成することができます。
    $ bundle exec yard doc
    $ open doc/index.html
### REST APIドキュメントの生成
REST APIのドキュメントは`api_doc/api.md`にAPI Blueprintの書式で記述します。
API BlueprintファイルをもとにHTMLファイルを生成するにはaglioというnpmモジュールを利用します。
aglioは`package.json`に記述済みなので、以下のコマンドでインストールすることができます。
    $ npm install
aglioのインストールが完了したら、以下のコマンドを実行することでHTMLファイルが生成されます。
    $ npm run build_apidoc
    $ open api_doc/api.html
`build_apidoc`コマンドは`package.json`に定義されています。
REST APIドキュメント(`api_doc/api.md`)を更新した際は、上記コマンドでHTMLファイルも更新してリポジトリにpushするようにします。
### リリース
[Railsアプリのデプロイ手順 · serverworks/kanrinmaru Wiki](https://github.com/serverworks/kanrinmaru/wiki/Rails%E3%82%A2%E3%83%97%E3%83%AA%E3%81%AE%E3%83%87%E3%83%97%E3%83%AD%E3%82%A4%E6%89%8B%E9%A0%86)
### コード品質のチェック
CoffeeScriptのコード品質は以下のコマンドでチェックすることができます。
    $ bundle exec coffeelint app/assets/javascripts
ルールについての設定は`coffeelint.json`で行われています。設定内容については[http://www.coffeelint.org/#options](http://www.coffeelint.org/#options)を参照してください。
SCSSのコードは以下のコマンドでチェックすることができます。
    $ bundle exec scss-lint
対象ファイルおよびルールについての設定は`.scss-lint.yml`で行われています。
設定内容については[https://github.com/brigade/scss-lint](https://github.com/brigade/scss-lint)を参照してください。
### I18n文字列のチェック
[i18n-tasks](https://github.com/glebm/i18n-tasks)の`missing`タスクを利用することで、Rubyのコード内におけるI18n文字列の参照（`t("foo.bar")`等）に対応する値が`config/locales`以下に定義されているかどうかをチェックすることができます。
    $ bundle exec i18n-tasks missing
    Missing translations (13) | i18n-tasks v0.9.5
    ...
