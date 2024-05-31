---
title: "リモートプロセスのIrbセッションをローカルプロセスのRelineを使って操作できるirb-remoteというGemを作った"
date: 2024-05-31T23:33:10+09:00
draft: false
---

irb-remoteというGemを作ったので紹介します。

https://github.com/QWYNG/irb-remote

# irb-remoteとは
irb-remoteは、リモートプロセスのIrbセッションをローカルプロセスのRelineを使って操作できるようにするGemです。
プロセス間通信にはdRubyを使用しています。

![](/irb-remote-demo.png)

## irb-remoteを作成した背景
[green_day](https://github.com/QWYNG/green_day)というAtcoderのページをスクレイピングしてテストを書いてくれるGemを作ったのですが、IOをキャプチャするテストをする関係上、Binding#irbを使ってデバッグすることが厳しかったので、リモートプロセスのIrbを操作できる何かがあればいいなと思い作りました。

```ruby
  it 'test with "2 900\n"' do
    # abc150/A.rb内でirbを起動してもIO
    io = IO.popen('ruby abc150/A.rb', 'w+')
    io.puts("2 900\n")
    io.close_write
    expect(io.readlines.join).to eq("Yes\n")
  end
```

```ruby
 1) abc150/A.rb test with "2 900\n"
     Failure/Error: expect(io.readlines.join).to eq("Yes\n")

       expected: "Yes\n"
            got: "\nFrom: abc150/A.rb @ line 1 :\n\n => 1: binding.irb\n\nSwitch to inspect mode.\n2 900\n/home/qwyng/...0\e[m\n\e[1m  ^~~\e[m\n\n\tfrom <internal:prelude>:5:in `irb'\n\tfrom abc150/A.rb:1:in `<main>'\n\n"

       (compared using ==)

       # abc150/A.rb内でirbを起動すると、ただテストが失敗するだけでIrbを操作できない
       Diff:
       @@ -1,13 +1,25 @@
       -Yes
       +
       +From: abc150/A.rb @ line 1 :
       +
       + => 1: binding.irb
       +
       +Switch to inspect mode.
       +2 900
       +/home/qwyng/.rbenv/versions/3.2.2/lib/ruby/3.2.0/irb/workspace.rb:119:in `eval': /home/qwyng/green_day/abc150/A.rb:1: syntax error, unexpected integer literal, expecting end-of-input (SyntaxError)
       +2 900
       +  ^~~
       +
       +        from <internal:prelude>:5:in `irb'
       +        from abc150/A.rb:1:in `<main>'

     # ./abc150/spec/A_spec.rb:6:in `block (2 levels) in <top (required)>'
```

irb-remoteを使うと、リモートプロセスのIrbを操作できるので、上記のような問題を解決できます。

![](/demo-with-greenday.png)

# dRubyを使うという発想
dRubyを使ってリモートプロセスのIrbを操作するという発想は、[pry-remote](https://github.com/Mon-Ouie/pry-remote)や[irb_remote](https://github.com/iguchi1124/irb_remote)でもうすでに試されていることでした。
どちらもRelineには対応しておらず、今回はRelineを使いたかったので別のGemを作りました。


## この記事を書いた理由
勉強会でirb-remote的な話がなされいたっぽいので...
正直僕の実装はめちゃくちゃ実装なので、ちゃんとした人がちゃんと作ったらもっといいものができると思います。
`irb-remote`という贅沢な名前を使ってしまったので、もしアレだったらすぐ名前変えます!!!すんません!!!





