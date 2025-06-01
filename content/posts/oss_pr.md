---
title: "OSSへの貢献と、PRを送ることへの後ろめたさ"
date: 2025-06-01T15:19:04+09:00
draft: false
---

## OSSへの貢献
2月で大学院関連の作業が一段落して、暇だったのでOSSにいくつかPRを送った。ありがたいことにいくつかPRをマージしてもらった。どっちもそんなに大きな変更ではないが、久しぶりにちゃんとPRを送ってマージされたので嬉しかった。

- [Binding#irbがデフォルトで引数渡せなかったことを修正したやつ](https://github.com/ruby/ruby/pull/12796)
- [faradayのプロキシオプションがスレッドセーフではない問題の修正したやつ](https://github.com/lostisland/faraday/pull/1617)

一方で、自分のPRが迷惑なのではないかと不安になる感情を久しぶりに思い出した。この記事では、僕のOSSへのPRに対する後ろめたさをつらつらと書いていく。

## disclaimer
僕は著名なOSSのメンテナをやったことはない。僕が想像しているメンテナの苦労とかは全く的外れの可能性がある。そもそも人よりOSSへの貢献が多いわけでもない。なのでOSSへのPRに対する考えがそもそも浅い可能性がある。  
また、OSSへのPRを咎めたいわけではない。あくまで自分に対する批判意識が重要だと思うだけで、他人のPRを見てこの記事を書いているわけではない。

## PRを送ることによって生まれるコスト
rails/railsへのPRに対する含蓄深いコメントがある。いくつか引用する。
https://github.com/rails/rails/pull/13771#issuecomment-32746700

> Someone need to spend the time to review the patch. However trivial the changes might seem, there might be some subtle reasons the original code are written this way and any tiny changes have the possibility of altering behaviour and introducing bugs. (For example, in this case, do you know if self.name= and assign_names doesn't depend on controller_name being set? I don't – so I'd have to look it up and make sure. Also, there is a travis failure associated with this PR, it's probably a random failure, but I'd have to investigate to be sure). All of these work takes away time and energy that could be spent on actual features and bug fixes.

PRを送るこということは、誰かにレビューの手間をかけさせる行為だ。もちろんメンテナがPRをレビューするしないは自由だが、もしマージする可能性があるならしっかりレビューし、しかも自分で直さず、「こう直してほしい」というコミニュケーションまでしないとならない。（一旦マージして後で直しているメンテナの人もいるとどこかで聞いた）

この話は個人的にかなり身をつまされる話だ。自分のPRがそのままマージされることはまずありえない。メンテナの人がしっかり修正箇所を指摘してマージしてもらうことがほとんどだ。見ず知らずの誰かの未成熟なPRを（無料で！）丁寧にレビューするというコストは計り知れないものがある。  

昔、僕の小さなOSSに対してPRを送ってくれる人がいた。そのコードにはテストがなかったので、テストを書いてほしいとお願いした。その後、そのPRにコミットが積まれることはなかった。これは僕のメンタルにかなりダメージを与えた。「もっと丁寧な言い方はなかったか」「マージして自分でテストを書いてしまえば良かったのでは」といった後悔が頭をよぎった。ここで言いたいのはPRを介したコミニュケーションというのはかなりメンタルに負荷がかかるということだ。
PRを送るということは、技術的にもメンタル的にもコストが発生する行為なのは間違いない。

> It pollutes the git history. When someone need to investigate a bug and git blame these lines in the future, they'll hit this "refactor" commit which is not very useful.

これはリファクタリングのPRに対してのコメントで、今回の話の大筋からは外れるが、確かにそうだなと思う。  
仕事のPRでも正直全部スカッシュしてマージしたほうがいいのではと思うことが多い。レビューする時は細かいコミットは有効だが、一回マージしたらPR単位でまとめてあったほうがblameしやすい。

> It's awesome that you want to contribute to Rails, please keep the PRs coming! All we ask is that you refrain from sending these types of changes in the future (and read the contribution guide :). I hope you'll understand!

> P.S. I'm not picking on you – it's just that we seem to be getting more PRs in the cosmetic changes category, and so I wanted to take this chance to explain our position and have something I can link to in the future.

とても優しいコメントだ。rails/railsはPRがたくさん来て大変なOSSだと思うが、それでもPRを出してくれることにリスペクトを示している。PRを出す側も同じくらいのリスペクトを持ってPRを出すべきだと思う。

２つ目はOSSに関するスライド。先述のコメントと同様に素晴らしいスライドなのでぜひ見てほしい。  
[OSSで結果を出す方法 - Speaker Deck](https://speakerdeck.com/knu/ossdejie-guo-wochu-sufang-fa)

> 他人の書いたコードを、今後は自分のコードとして責任を持ち、ずっとメンテナンスしていくということ。
> 気軽にOKした変更も、洗礼となって次の要望を呼び込む

これもPRを出す側が意識すべきことだ。自分が出してしまったPRをずっとメンテするのはメンテナの人だ。少なくともきちんとテストされているべきだし、マージされないからと言って文句が言える立場ではない。
CONTRIBUTION.mdに書いてあることを守るのはもちろん、PRテンプレをよく読んだりするべきだろう。

> 「MySQLだけでなく、PostgresSQLでも動くようにしました!」
> 複数対応をずっと続けられるのか
> 固有の問題に対応できるのか
> 差異を将来に渡って吸収できるのか

実際、僕も自分の技術力を超えている内容のPRを出して、めちゃくちゃメンテナの人に迷惑をかけたことがある。そのPRは果たして世界のためにプラスだったのだろうか、それもとトータルマイナスだったのだろうか。今でもわからない。

## OSSへの貢献の動機
普段使っているOSSに〇〇というバグがある、直そう！というのが健全なPRの動機ではあると思う。
しかし、OSSにPRを送る動機はそれだけではないはずだ。単に暇つぶしのためだったり、面白いからだったり、承認欲求のためだったり、様々な動機があると思う。
僕がOSSに貢献する動機は、利用者が多いOSSに自分のコードがマージされると人類にとてつもない貢献をした気になって最高になるからだ。だから自分でもなんとかできそうなissueが立っていたりすると嬉しい。ある意味自分の欲求のためにOSSを利用している。こういう動機を自覚しているのでどうしてもPRを送る事にたいしてどこか後ろめたさを感じる。

## 結局どうするのか
もっとしっかりしたコードでもっとしっかりPRを送ろう！あと別に自分でOSSを作ってもいいんだよ！ということを自分に言い聞かせたい。









