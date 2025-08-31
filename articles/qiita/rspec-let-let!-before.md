---
title: 【RSpec】"let"と"let!"とbeforeの違いと実行順序
tags: Ruby Rails RSpec
author: kikikikimorimori
slide: false
---
### はじめに
rspecのテストコードを書くために"let"と"let!"の違いを調べていたところ、今度は"let!"と"before"の違いや実行順序に関して気になったのでまとめてみました。
### "let"とは
rspecには"let"というメソッドがあります。
これは呼び出された時にはじめてデータを読み込む「遅延読み込み」を行う機能です。

```rb:
  describe 'let'do
    let(:user) { build(:user) }

    it 'userインスタンスが有効であること' do
      expect(user).to be_valid 
             ※↑ここではじめて読み込まれる
    end
  end
```

上記のように呼び出された時にはじめてデータを読み込むので、無駄にデータベースに問い合わせてしまってテストが遅くなるといったことを予防してくれます。

ちなみに上記のテストはパスします。

逆に下記のテストは通りません。

```rb:
  describe 'let'do
    let(:user) { create(:user) }
    let(:user_task) { create(:task, user_id: user.id, title: 'test_title') } 

    it 'userがtaskを持っていること' do
      expect(user.tasks.first.title).to eq 'test_title' 
    end
  end
```
このテストでは、「user」は呼び出されているのでテストデータが読み込まれますが、
「user_task」は呼び出されていないので、このままでは「let(:user_task)　・・・・」の部分は、ただの無駄な記述になってしまいます。

### "let!"とは
先程のテストをパスさせるために使うのが、「let!」メソッドです。
こちらは"let"と違い、テストが実行される前に自動的にテストデータが読み込まれます。

なので、先程の「let(:user_task)」の部分を「let!」にします。

```rb:
  describe 'let'do
    let(:user) { create(:user) }
    let!(:user_task) { create(:task, user_id: user.id, title: 'test_title') } 

    it 'userがtaskを持っていること' do
      expect(user.tasks.first.title).to eq 'test_title' 
    end
  end
```

上記のテストではまず、taskインスタンスの入った変数「user_task」が作成されます。

続いて、expect内のテストが実行されるので、今回はテストがパスします。


### beforeとは
beforeは、「let!」とほぼ同じと思っていただいて問題ありません。、、、、多分。

主な違う点は、インスタンス変数の定義以外にも使用できることです。

例えば、ログイン後の動作をテストしたい場合、

```rb:
  describe 'after login' do
    let(:user) { create(:user) }
    before { login(user) } #loginメソッドを定義している前提
    
    it 'is success to edit user' do
      visit edit_user_path(user)
      fill_in "Email", with: "edit_user@example.com"
      click_button "Update"
      expect(page).to have_content 'User was successfully updated.'
      expect(current_path).to eq user_path(user)
    end
  end
```
この長さのテストコードではあまり意味がありませんが、通常はログイン後に確認したい挙動はたくさんあるはずなので、
このようにbeforeで定義することで、コードのリファクタリングにもなります。

### "let!"と"before"の実行順序
先程の説明で「let!」と「before」はほぼ同じと言いましたが、実行順序はどうなるのでしょうか？

結論から言うと、定義した順番で実行されます。

例えば以下のような場合、

```rb:
let!(:user_1) { create(:user) }
before { user_2 = create(:user) }
let!(:user_3) { create(:user) }
```
上から順にuserインスタンスが作成されていきます。

### 結局どれ使えばいいの？
これに関しては状況次第です笑

簡単にまとめると、

・"let"　←　テストで何回か使うインスタンス変数がある時
・"let!"　←　テストのブロック内で毎回使うインスタンス変数がある時
・"before"　←　テストのブロック内で毎回適用させたいメソッドなど（インスタンス変数以外）がある時

あくまで参考程度ですがこんな感じの基準でいいと思います笑

間違いなどあればコメントください！


### 参考
[「let・let!・before」の違いと実行順序（RSpec） - ryotaku's Tech Blog](https://www.ryotaku.com/entry/2019/09/10/000000)
[rspec 「let」と「let!」の違い](https://qiita.com/clbcl226/items/3db2189f75747b4af2d3)
