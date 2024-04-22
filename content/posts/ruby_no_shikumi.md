---
title: "Rubyのしくみを読んだ"
date: 2024-04-22T21:44:50+09:00
draft: true
---

RubyKaigiも近いということで積読していた[Rubyのしくみ　Ruby Under a Microscope
](https://tatsu-zine.com/books/ruby-under-a-microscope-ja)を読んだので感想を書きます。

全部で12章ある本なんですが、自分の中での区切り順で書きます。

## 1章, 2章 パースとコンパイル
字句解析、構文解析、コンパイルについて書かれている。
正直、ここが一番よくわかってない。

- 字句解析と構文解析という概念がある
  - 字句解析は文字列をトークンに分解する
  - 構文解析はトークンを構文木に変換する
  - 字句解析のステートマシンとか構文木の話を詳しく知りたい人は[アンダースタンディング コンピュテーション](https://www.oreilly.co.jp/books/9784873116976/)を読もう！Rubyからコンピューター最センスが学べちまうんだ!(僕は積んでいます)
- コンパイル
  - 

## 3章, 4章　VMの動作
VMの動作について書かれている。
- 内部スタックとコールスタック
  - 院で研究しているプログラムの操作的意味論の[サンプル](https://kframework.org/k-distribution/pl-tutorial/2_languages/2_kool/1_untyped/kool-untyped/
)も環境スタックとコールスタックを２つもつ実装だった。
  



