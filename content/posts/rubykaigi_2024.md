---
title: "RubyKaigi 2024 感想"
date: 2024-05-21T00:00:00+09:00
draft: false
---

RubyKaigi 2024に参加したので気になったセッションの感想を書いたりします。

## Unlocking Potential of Property Based Testing with Ractor @ohbarye
- Ractorを使ったプロパティベーステストの話
- SPINとかLTLとかJAISTでも学んだので面白かった
- よくあるアプリの「IOが一番のボトルネックであんまりRactorとかThreadかどうか関係ない」っていう要素をクリアしていて良い仮説だな〜と思った
- net/httpがRactorに対応してなくて辛いのは本当にそう
  - 並列処理においてスクレイピングって結構思いつくところだけど、net/httpがRactorに対応してなくて辛かった思ひ出がある
  - GreenDayをRactorで書き直そうと思ったけど、net/httpが対応してないので断念した
  - その時に発見したレポジトリなんだけどRactorで共有不可能なオブジェクトをメインRactorでプロキシするみたいなGemはすごい発想だなと思った
    - https://github.com/dazuma/ractor-wrapper

## Exploring Reline: Enhancing Command Line Usability @ima1zumi
- Relineの話
- Relineってまじで便利すぎますよな〜変換候補がでたりYARDのドキュメントを見れたり
- GNU Readlineを目指しているってすごいことですよ
  - Rubyでコマンドラインエディターの実装が読めるって神すぎる
  - IRBのRelineInputMethodの実装だとpromptとかterminationのcheckとかindentとか全部procで実装してあって面白かった
  - [irb-remote](https://github.com/QWYNG/irb-remote)っていうまさにRelineを使いたいがためのGemを作っているのでいい感じになったらどこかで公開したい

## Ractor Enhancements, 2024 @ko1
- Ractorの話
  - Ractorのyield/takeとsend/receiveどっちも書けるのがRubyを体現しててすごいと思っているのでRactorの話は好き
  - requireがRactorに対応してないのは辛い本当にそれ
  - 子Ractorのkernel#requireをMainRactorでプロキシする方法はissue内でも紹介されていたけど、言語の機能として実装するには色々魔改造されがちなKernel#requireをどうするかの問題があるのだなと思った
  - タイムアウトも言われてみるとErlangとかはどうしているんだろうと思った。
  - グリーンスレッドとM:Nスレッドの違いがまだはっきりとわかっていなくて、N個のOSスレッドに対してM個のRubyスレッドを作るのがM:Nスレッドで、グリーンスレッドはユーザレベルスレッドだと思っている
    - でもRactorは結局GVLを外せるユーザレベルスレッドということなのかな？
    - [Rubyの並列並行処理のこれまでとこれから](https://techlife.cookpad.com/entry/2023/08/31/152511)これを読んでもそんなにわからなかったが、またどこかで勉強しようという気持ちになった

## アフターパーティー
- 飲み会がド級の苦手、ド苦手なのだが、今回は勇気を振り絞ってオフィシャルのアフターパーティーに参加した
- IVRyさんとたまたま同じ席になり、プロダクトのコードについて話した
  - 僕は今5~6年前に書かれたコードに感謝しながらプロダクトコードを書いているのだが、いずれIVRyのみなさんが書いたコードもそうなるんだろうと思うと感慨深かった
  - 丁寧なコミットメッセージやPR文やドキュメントがいかに助かるかという話をした気がする
  - ちなみに私のコミットメッセージとPRは...

## まとめ
RubyKaigiはやっぱり楽しいですね。運営の皆様、スポンサーの皆様、発表者の皆様、参加者の皆様、ありがとうございました。また来年も参加したいです。  
どの発表も自分が日々使っている道具がどんな方によってどう作られているのかを知ることができて、とても楽しかったです。

