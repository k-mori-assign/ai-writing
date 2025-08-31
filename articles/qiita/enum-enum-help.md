---
title: enumとenum_helpの使い方【rails】
tags: Ruby Rails enum
author: kikikikimorimori
slide: false
---
### はじめに
`enum`と`enum_help`について

### enumって何？？
モデルで数値のデータ型で定義しているカラムを、
文字列型として使えるようにすることができます。

例えば、0は男性、1は女性みたいな感じです。

### 定義方法
enumはモデルに定義する必要があります。
定義の仕方は主に2つあります。

・名前だけを定義する方法
・名前とそれに対応する数値を定義する方法

では、1つずつ見ていきましょう！

####  1.名前だけを定義する方法
まず、前提としてUserモデルのroleカラムがあるという前提で説明していきます。


```rb:models/user.rb
class User < ApplicationRecord
  enum role: [ :general, :admin ]
end
```

上記のように定義します。

「role」の部分がカラム名です。
その横に配列として、使いたい名前を定義していき、左から順番に0から数字が紐づけられていきます。

上の例でいうと、
「general: 0, admin: 1」という風になります。

ちなみ後から下記のように定義を追加した場合、

```rb:models/user.rb
class User < ApplicationRecord
  enum role: [ :general, :writer, :admin ]
end
```

「general: 0, writer: 1, admin: 2」と言う風になります。

つまり、後から定義したとしても、数字は前から順に割り振られていきます。
なので、仮にすでにデータベースに「admin」が保存されていた場合、
それが「writer」ということになってしまいます。

####  2.名前とそれに対応する数値を定義する方法

```rb:models/user.rb
class User < ApplicationRecord
  enum role: { general: 0, admin: 1 }
end
```

こちらの方法は、明示的に名前と対応する数字を定義します。

ちなみに、こちらの方法は後から定義を追加する場合は
下記のように数字を明示的に指定する必要があります。

```rb:models/user.rb
class User < ApplicationRecord
  enum role: { general: 0, writer: 2, admin: 1 }
end
```
なので、先程のようにデータが途中から変わってしまうというようなことは起こり得ません。
明示的に指定したほうがわかりやすいので私はこちらの方法を推奨します。

また、一番最初に定義している値、つまり「general: 0」の値をデフォルトとしてモデルに定義すると、より良いです。

```rb:マイグレーションファイル
class AddRoleToUsers < ActiveRecord::Migration[5.2]
  def change
    add_column :users, :role, :integer, null: false, default: 0
  end
end
```

### コンソールで使ってみる
```rb:コンソール
irb(main):001:0> User.roles    ※定義済のカラムを参照
=> {"general"=>0, "admin"=>1}  ※ハッシュ形式で保存されている

irb(main):002:0> User.roles[:general] ※ハッシュなので一項目だけ参照することも可能
=> 0
irb(main):003:0> User.roles[:admin]
=> 1

irb(main):004:0> user = User.first

irb(main):005:0> user.general?  ※このような便利なメソッドも使えるようになります。
=> false
irb(main):006:0> user.admin?
=> true

irb(main):006:0> user.general!  # roleをgeneralに変更する

irb(main):006:0> User.general  # roleカラムがgeneralのデータを取得する
                               # User.where(role: :general)と同じ挙動
```

### enum_helpって何？？

enum_helpとはenumで定義した値をi18n化させることができるgemです！

現段階では下記のようになります。

```rb:コンソール
irb(main):007:0> user.role
=> "admin"

※これを日本語にしたい！
```

### 導入方法

導入方法は簡単！
enum_helpというgemを追加して、localeに翻訳を設定するだけです！

```rb:Gemfile
gem 'enum_help'
```
```
$ bundle install
```
```rb:config/locales/activerecord/ja.yml
ja:
  enums:
    user:
      role:
        general: 一般
        admin: 管理者
```
```rb:コンソール
irb(main):007:0> user.role
=> "admin"

irb(main):008:0> user.role_i18n  ※カラム名のあとに「_i18n」をつけて呼び出す
=> "管理者"
```

enum_helpを導入すると、便利なヘルパーも追加されます。

```rb:コンソール
irb(main):011:0> User.roles
=> {"general"=>0, "admin"=>1}

irb(main):012:0> User.roles_i18n
=> {"general"=>"一般", "admin"=>"管理者"}

irb(main):013:0> User.roles_i18n.invert
=> {"一般"=>"general", "管理者"=>"admin"}  ※キーとバリューが入れ替わる！
```
このinvertメソッドはセレクトボックスを使う時やransackを利用した時などに有効です！

※間違いなどあればコメントください！

