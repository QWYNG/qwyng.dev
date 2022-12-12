---
title: "Railsでよく使用されるマルチテナントGemのテナントの初期化について"
date: 2022-12-07
draft: false
---


このエントリは、[SmartHR Advent Calendar 2022](https://qiita.com/advent-calendar/2022/smarthr)の7日目です。

SmartHRではマルチテナントの実現のために [activerecord-multi-tenant](https://github.com/citusdata/activerecord-multi-tenant)というGemを使っているのですが、そのGemを調査したときに気づいたことを書きたいと思います。

### TL;DR
- `activerecord-multi-tenant`と`ActsAsTenant`はリクエストごとにテナントを初期化するタイミングが違うよ。
- とくにrequest specでは誤ったテストの原因になりかねないので、テナントが初期化されるタイミングを理解して実際のアプリケーションの動作になるだけ近づけよう。

### テナントのリセットタイミングの再現コード
Railsにおいてマルチテナントを行う際に有力な選択肢として
- [ActsAsTenant](https://github.com/ErwinM/acts_as_tenant)
- [activerecord-multi-tenant](https://github.com/citusdata/activerecord-multi-tenant)

あたりが有力な選択肢になるかと思われます。
（[Apartment](https://github.com/rails-on-services/apartment)も有名ですが先述の２つのGemと異なりマルチスキーマでマルチテナントを構成するGemで検証が間に合わなかったので言及しません :bow: )  
どちらのGemもリクエストごとに設定されているテナントを初期化してくれる機能が提供されていますが、その初期化をするタイミングが微妙に異なっています。  
↓サンプルコードです。

```ruby
require 'bundler/inline'

gemfile(true) do
  source 'https://rubygems.org'

  gem 'rails', '7.0.4'
  gem 'sqlite3', '1.5.4'
  gem 'activerecord-multi-tenant', '2.1.6', require: false
  gem 'acts_as_tenant', '0.5.3'
  gem 'rspec-rails'
end

require 'action_controller/railtie'
require 'active_record'
ActiveRecord::Base.establish_connection(adapter: 'sqlite3', database: ':memory:')

require 'activerecord-multi-tenant'

class App < Rails::Application
  routes.append do
    get '/tenant_test' => 'tenant#test'
  end
end

class TenantController < ActionController::API
  def test
    puts "ActsAsTenant.current_tenant:  #{ActsAsTenant.current_tenant}"
    puts "ActsAsTenant.test_tenant:  #{ActsAsTenant.test_tenant}"
    puts "MultiTenant.current_tenant:  #{MultiTenant.current_tenant}"

    render json: { hello: :world }
  end
end

App.configure do
  config.hosts.clear
end

require 'acts_as_tenant/test_tenant_middleware'

Rails.application.configure do
  config.middleware.use ActsAsTenant::TestTenantMiddleware
end

App.initialize!

require 'rspec/rails'

RSpec.describe 'Test', type: :request do
  it 'first request' do
    ActsAsTenant.current_tenant = 'Tenant set in spec'
    ActsAsTenant.test_tenant = 'Tenant set in spec'
    MultiTenant.current_tenant = 'Tenant set in spec'

    get '/tenant_test'

    expect(response.status).to eq(200)
  end

  it 'second request' do
    get '/tenant_test'

    expect(response.status).to eq(200)
  end
end
```

実行結果

```
⋊> ~/acts_as_tenant_sandbox rspec -f d script.rb      
Fetching gem metadata from https://rubygems.org/...........
###(中略)###

Test
ActsAsTenant.current_tenant:  Tenant set in spec
ActsAsTenant.test_tenant:  
MultiTenant.current_tenant:  
  first request
ActsAsTenant.current_tenant:  
ActsAsTenant.test_tenant:  
MultiTenant.current_tenant:  
  second request

Finished in 0.00789 seconds (files took 1.34 seconds to load)
2 examples, 0 failures
```

実行結果からわかる通り`ActsAsTenant.current_tenant`はテストコード内で設定したテナントがrequest specの機能で呼び出されたアプリケーションの中でも初期化されずに引き継がれていることがわかります。
この問題(？)については`ActsAsTenant`のREADMEにて記載があります。
>If you set the `current_tenant` in your tests, make sure to clean up the tenant after each test by calling `ActsAsTenant.current_tenant = nil`. Integration tests are more difficult: manually setting the `current_tenant` value will not survive across multiple requests, even if they take place within the same test. This can result in undesired boilerplate to set the desired tenant. Moreover, the efficacy of the test can be compromised because the set `current_tenant` value will carry over into the request-response cycle.

また対処の方法としてサンプルコードでも使用していた`ActsAsTenant.test_tenant`の使用を推奨する記載もREADMEにあります。

### なぜ初期化するタイミングが違うのか

なぜ初期化するタイミングが違うのかというと`ActsAsTenant`と`MultiTenant`ではテナントの保存に使用しているクラスが異なっているからです。

`ActsAsTenant`は`RequestStore`を[使っています。](https://github.com/ErwinM/acts_as_tenant/blob/v0.5.3/lib/acts_as_tenant.rb#L59)  
- https://github.com/steveklabnik/request_store

対して`MultiTenant`は`ActiveSupport::CurrentAttributes`を[使っています。](https://github.com/citusdata/activerecord-multi-tenant/blob/a6c8c2d75230ac6ebb22a609d6f7fc57e5f250ee/lib/activerecord-multi-tenant/multi_tenant.rb#L60)

- https://github.com/rails/rails/blob/v7.0.4/activesupport/lib/active_support/current_attributes.rb

どちらもThreadクラスを利用しつつリクエストごとに値を初期化してくれる場所を提供してくれるクラスですが、初期化を設定する方法が異なっています。

`RequestStore`はRackに[値を初期化するミドルウェア](https://github.com/steveklabnik/request_store/blob/v1.5.1/lib/request_store/middleware.rb)を追加しているのと、RailsのExecuterの`to_complite`コールバックに[値を初期化する動作が追加](https://github.com/steveklabnik/request_store/blob/f79bd375e88f434428b876dbb5c8a51b569712aa/lib/request_store/railtie.rb#L10) されています。  
Executerのコールバックについての細かい解説は[Railsガイド](https://railsguides.jp/threading_and_code_execution.html#executor)にあります（いつもありがとうございます！）。

ミドルウェアの中身は↓のようになっており、`RequestStore`はリクエストを処理した後にのみ値を初期化していることがわかります。

```ruby
def call(env)
  RequestStore.begin!

  status, headers, body = @app.call(env)

  body = Rack::BodyProxy.new(body) do
    RequestStore.end!
    RequestStore.clear!
  end
  ...
```

対して`ActiveSupport::CurrentAttributes`ではRails のExecutorのコールバックである`to_run`と`to_complete`に[値を初期化する動作を追加しています。](https://github.com/rails/rails/blob/v7.0.4/activesupport/lib/active_support/railtie.rb#L41)  
先述のガイドにものっていますが、`to_run`コールバックはアプリケーションの実行前に呼び出されるコールバックです。

↓は超ざっくりイメージです。
![](/reset.png)


### 初期化されるタイミングの違いで起こりうること
request specで使えるようになる`get`等のメソッドはテストと同じスレッドで擬似的にRackアプリ全体を呼び出すため、`RequestStore`を使用している`ActsAsTenant.current_tenant`はテスト内で設定した`current_tenant`の値が初期化されずに引き継がれます。  
request specを統合テストと考えると事前にテナントが固定されたままテストするのはあまりよくないので[リクエストを処理する前に値を初期化してくれるミドルウェアに対応している](https://github.com/ErwinM/acts_as_tenant/tree/v0.5.3#testing)`ActsAsTenant.test_tenant`を使用したほうが良いでしょう。  
実は `activerecord-multi-tenant`も古いバージョンでは`RequestStore`を使用しており、`ActsAsTenant.current_tenant`と同様にrequest spec内で固定したテナントが`get`等で呼び出すアプリケーション内で初期化されず引き継がれるようになっていました。  
request specを書く際にはテストのためにデータを準備するコードでテナントを固定する箇所は発生すると思いますので、
テナントがスレッド内でどのように固定と初期化されているのか理解しておくことで間違ったテストを書く可能性が減らせると思います。
