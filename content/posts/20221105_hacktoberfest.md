---
title: "Hacktoberfest 2022で建てたOSSへのPR"
date: 2022-11-05T14:56:26+09:00
draft: false
---

有給消化の一ヶ月間好き放題してたらRubyの全てを忘れてしまったのでHacktoberfestの機会にいくつかPRをだした。  
なおHacktoberfest対象レポジトリ以外にもPRは建ててます。

### Pundit
https://github.com/varvet/pundit/pull/745

rubocop-rspecには外部Gemが追加したRspecのDSLをLintの対象にできる[機能](https://docs.rubocop.org/rubocop-rspec/third_party_rspec_syntax_extensions.html
)がある。  

PunditのRspec用DSLである`permissions`を対応させた。good first issueという感じですぐ作業が完了したので特にいうことはなし。  
### bake-test-external
https://github.com/ioquatix/bake-test-external/pull/2

開発中のGemに依存している別のGemのテストを、開発中のGemを使って実行できるGem。Samuel Williamsさん製。  
後述するPRの作業中に必要になったのでPRを建てた。  

このGemの仕組みとしてテストしたいGemのレポジトリをcloneしてきてGemfileにローカルのGemを参照させるよう加筆する仕組みになっているのだけれど、gemspecで参照されることのみ想定していたのかGemfileに開発中のGemの名前が書いてあるとGemfileに同じ名前が２つのってしまっていたので修正した。  
PRの内容がいちいち正規表現合成してたり良くないコードだったんだけど  
`I'm generally okay with this, but I'm a little concerned it's fragile. However, it's not critical it breaks.`  
というオブラートに包んだコメントもらった後にすぐマージしてくれた。なお上からちゃんと修正してくれていた。押しかけコミット押し付け太郎になってしまい反省しています。

### omniauth
https://github.com/omniauth/omniauth/pull/1095

ここからはまだマージされてないPR。  
bake-test-externalについて知ったのはこのPRが理由。
bake-test-externalを使ってomniauthのCIにdeviseのテスト実行を追加するPR。issueにgood first issueラベルが貼ってあったので取り組んでみた。確かにomniauthの変更でdeviseがバグってないか確認できたら便利そうではある。  
マージされるかはわからないが、メンテナーの人が最近忙しいみたいなことをレポジトリのどこかで書いていた気がするのでクローズされるにしろマージされるにしろ時間かかりそう

### Puma
https://github.com/puma/puma/pull/3006

Twitterでメンテナーの方が宣伝してたので便乗してPRを建てた。
<blockquote class="twitter-tweet"><p lang="en" dir="ltr">An opportunity for Puma contribution - write some tests for an already-completed feature <a href="https://t.co/3cQREdtJH7">https://t.co/3cQREdtJH7</a></p>&mdash; Nate Berkopec (@nateberkopec) <a href="https://twitter.com/nateberkopec/status/1585862608640352256?ref_src=twsrc%5Etfw">October 28, 2022</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

完全に人の褌で相撲をとっている状態である。  
自分でやったのはSymbolとString変換したりテストちょろっと足したぐらい。  
このPRはメンテナーの方からスタンプとラベル貰えたのですぐにクローズされることはなさそう。
`sd_notify`って概念初めて知ったし勉強になった。

### あとがき
こいつgood first issueばっかやっとるな。