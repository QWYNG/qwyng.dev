---
title: "Rubyのパターンマッチをこうやって使いたい"
date: 2025-07-19T09:54:04+09:00
draft: false  
---

Rubyのパターンマッチ、最高です。  
ですが、最近、以下のような使い方をしたいんですができなくて困っています。

```ruby
ages = [40, 60, 75]
users = [{name: "佐藤", age: 74}, {name: "鈴木", age: 75}]
pp users.any? { |user| user in { name: "鈴木", age: ^ages } }
#=> 鈴木さんが40歳か60歳か75歳ならtrueになってほしい
```

このコードはRuby3.4.5でfalseを標準出力します。`{ name: "鈴木", age: [40, 60, 75] }` の人はいないからです。  
こういう複数パターンを論理和っぽく組み合わせたい場合、RubyのパターンマッチングにはAlternativeパターンという方法が提供されています。

```ruby
users = [{name: "佐藤", age: 74}, {name: "鈴木", age: 75}]
pp users.any? { |user| user in { name: "鈴木", age: 40 | 60 | 75 } }
# => true
```

これはこれでいいんですが、これだと条件を動的に変更することができません。  
実務だと、こういう条件をArrayで管理して分岐で可変にしがちです。

```ruby
ages = [40, 60, 75]
ages << 70 if some_reason
```

このArrayをそのままAlternativeパターンに組み込めたら最高なんですが、僕の調べた限り、現状のRubyにはArray(もしくはEnumerable)をそのままAlternativeパターンに組み込むやり方がないです。
調べた限りではAlternativeパターンに対応する言語は多くあるものの、配列をAlternativeパターンにするような構文を持つ言語は無いっぽい。そもそもやりたいことの筋が良くないのかもしれませんね。  

ガード節を使ってそれっぽいことはできるんですが、もっとCOOLにしたいんですよね

```ruby
ages = [40, 60, 75]
users = [{name: "佐藤", age: 74}, {name: "鈴木", age: 75}]

pp users.any? do |user|
  case user
  in { name: "鈴木", age: age } if ages.include?(age)
    true
  else
    false
  end
end
#=> true
```

ちなみに一行パターンマッチではなぜかうまくガード節に変数渡せなかった..

```ruby
ages = [40, 60, 75]
users = [{name: "佐藤", age: 74}, {name: "鈴木", age: 75}]
pp users.any? { |user| user in name: "鈴木", age: age if ages.include?(age) }
#=> false if内のageはnilになる
```

## 解決策 ===を良い感じにしたオブジェクトに変換する
カスタムクラスを噛まして`===`を良い感じにします。

```ruby
class ArrayAlt
  def initialize(array)
    @array = array
  end

  def ===(other)
    @array.include?(other)
  end
end

class Array
  def to_alt
    ArrayAlt.new(self)
  end
end

ages = [40, 60, 75]
users = [{name: "佐藤", age: 74}, {name: "鈴木", age: 75}]
pp users.any? { |user| user in  name: "鈴木", age: ^(ages.to_alt) }
#=> true
```

だいたいこれで良さそうです。  
ただ、Arrayをオープンクラスするのもなんだか嫌じゃないですか。僕はPRにパターンマッチをサラッといれてドヤりたいだけなんです！   
こんな感じの構文で良い感じになって欲しいんです。

```ruby
ages = [40, 60, 75]
users = [{name: "佐藤", age: 74}, {name: "鈴木", age: 75}]
# *^ages で展開されてAlternativeパターンになって欲しい
# なんか他と被らない良い感じのsyntaxはないものか
pp users.any? { |user| user in { name: "鈴木", age: *^ages } } 
```

誰かCOOLな方法を知っている人がいたら教えてください。  












