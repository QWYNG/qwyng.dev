---
title: "デバッガを活用してブレイクポイント貼り直し作業をやめよう"
date: 2025-11-02T21:23:18+09:00
draft: false
---

筆者は仕事でRuby on Railsを用いてウェブアプリケーションを開発している。  
愚かにもTDDはせず、まず動きそうなコードを書いてからテストを書き、その後デバック作業を行っている。  
その際、「ブレイクポイントを何度も貼り直して再実行するの面倒だな..」と感じていた。  
今回、その解決策に気付いたのでブログを書いていく。

## ブレイクポイント貼り直し作業
ここでは、パラメーターから作成するレコードを判定するロジック[^1]を持つアクションをもとに説明する。

```ruby
class SampleController < ApplicationController
  def create
    @post = if something_ok?(post_params)
      Post.new(post_params)
    else
      AnotherPost.new(post_params)
    end

    @post.save!

    redirect_to @post
  end

  private

  def something_ok?(params)
    # 意図的にロジックを挟んでいる
    something = params[:something].split(' ').reverse
    something.include?('ok')
  end
end
```

ここで、このエンドポイントが201を返すことを検査するテストが失敗したとする。

```ruby
it 'should return http status code 201' do
  post :create, params: { post: { title: 'title', something: complex_object }

  expect(response).to have_http_status(201)
end
# => expected: 201 got: 503
```


二分探索の最初として、メソッドのだいたい真ん中の位置にデバッガ[^2]の起動のためのブレイクポイントを設置し、@post.save!の挙動を確認する。

```diff
class SampleController < ApplicationController
  def create
    @post = if something_ok?(post_params)
      Post.new(post_params)
    else
      AnotherPost.new(post_params)
    end

+   debugger
    @post.save!

    redirect_to @post
  end

  private

  def something_ok?(params)
    something = params[:something].split(' ').reverse
    something.include?('ok')
  end
end
```

ここで、Postのオブジェクトに対してsave!が実行されてほしい所で、AnotherPostからエラーが起きてしまったとする。  
そうなるとsomething_ok?の挙動を確認したくなる。ただし、something_ok?の返り値だけでは十分ではない。something_ok?にはロジックが含まれているので、このメソッドの内部でもデバッガを用いて詳細な挙動を確認したくなる。  
ここがこの記事の問題の核心である。二分探索でデバックする場合、最初からいい感じの位置にブレイクポイントは設置できない事が多い。つまり、**間違っている箇所の位置が特定できていない状態からデバックをする場合、最終的に確認したい位置にデバッガーを最初からおけるケースは稀である。** ということを言いたい。  

こういう場合、筆者はブレイクポイントを手動で貼り直し、テストを再実行していた。

```diff
class SampleController < ApplicationController
  def create
    @post = if　something_ok?(post_params)
      Post.new(post_params)
    else
      AnotherPost.new(post_params)
    end

-   debugger
    @post.save!

    redirect_to @post
  end

  private

  def something_ok?(params)
    something = params[:something].split(' ').reverse
+   debugger
    something.include?('ok')
  end
end
```

しかし、この方法にはいくつか問題がある。
1. something_okの内部のブレイクポイントが適切かもわからない。更に深い場所までブレイクポイントで確認したくなるかもしれない。その場合、また手動でブレイクポイントを移動することになる。
2. テストを再実行するのは手間である。特にコントローラーの含めたテストだと、準備も高コストな場合が多い

ブレイクポイントを貼り直して、コードを再実行して..を行っている間に、筆者はSlackの巡回を開始するだろう。

## 解決策: ブレイクポイント設定機能を活用する
Rubyのdebug gemをはじめ、多くのモダンなデバッガにはデバッグ中に新たにブレイクポイントを仕込める機能がある[^3]。
この新たにブレイクポイントを仕込める機能を使えば、何度でもブレイクポイントを貼り直せるし、コード全体を再実行する必要もなくなる。

```bash
(rdbg) b SampleController#something_ok?    # break command
#0  BP - Method  SampleController#something_ok? at /Users/qwyng/sandbox_rails/app/controllers/sample_controller.rb:21
(ruby) something_ok?(post_params)
[17, 25] in ~/sandbox_rails/app/controllers/sample_controller.rb
    17|   def post_params
    18|     params.require(:post).permit(:title, :something)
    19|   end
    20|
    21|   def something_ok?(params)
=>  22|     something = params[:something].split(' ').reverse
    23|     something.include?('ok')
    24|   end
    25| end
=>#0    SampleController#something_ok?(params=#<ActionController::Parameters {"title" =...) at ~/sandbox_rails/app/controllers/sample_controller.rb:22
  #1    SampleController#create at (rdbg)//Users/qwyng/sandbox_rails/app/controllers/sample_controller.rb:1
  # and 91 frames (use `bt' command for all frames)

Stop by #0  BP - Method  SampleController#something_ok? at /Users/qwyng/sandbox_rails/app/controllers/sample_controller.rb:21
(rdbg) n    # next command
[18, 25] in ~/sandbox_rails/app/controllers/sample_controller.rb
    18|     params.require(:post).permit(:title, :something)
    19|   end
    20|
    21|   def something_ok?(params)
    22|     something = params[:something].split(' ').reverse
=>  23|     something.include?('ok')
    24|   end
    25| end
=>#0    SampleController#something_ok?(params=#<ActionController::Parameters {"title" =...) at ~/sandbox_rails/app/controllers/sample_controller.rb:23
  #1    SampleController#create at (rdbg)//Users/qwyng/sandbox_rails/app/controllers/sample_controller.rb:1
  # and 91 frames (use `bt' command for all frames)
(rdbg) something
["test", "ok"]
```

## まとめ
デバッガのブレイクポイントを追加する機能を使うと、コードを再実行せずにブレイクポイントを貼り直してデバッグを継続できる。  
二分探索デバッグでなんどもブレイクポイントを貼り直しているそこのアナタ、この機能を使おう！



[^1]: 将来、このロジックは肥大化し、フォームオブジェクトみたいな専用のレイヤが設けられるであろう。
[^2]: debug gem: https://github.com/ruby/debug?tab=readme-ov-file
[^3]: Rubyのdebug gem: https://github.com/ruby/debug?tab=readme-ov-file#breakpoint  
Pythonのpdb: https://docs.python.org/ja/3.13/library/pdb.html#pdbcommand-break  
gdb: https://sourceware.org/gdb/current/onlinedocs/gdb.html/Break-Commands.html#Break-Commands
