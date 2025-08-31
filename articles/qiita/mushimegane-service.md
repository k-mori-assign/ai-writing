---
title: 【個人開発】家の中にときどき出る不快な虫を検索するサービス「むしめがね」をリリースしました。
tags: Rails Ruby 個人開発 ポートフォリオ 初心者
author: kikikikimorimori
slide: false
---
### はじめに
はじめまして。森と申します。

この度、個人サービス「むしめがね」をリリースしました！
サービスに関して記事にしましたので、ぜひご一読いただけたら嬉しいです！

アプリのURL：[~~https://mushimegane.fun/~~](https://mushimegane.fun/)
※現在停止しています。

※記事内でゴキブリの画像がでてきます。

### サービス概要
「むしめがね」は、家の中にときどき出て来る不快な虫を検索できるサービスです。

全体を通してシンプルなデザインにしています。
トップページには興味をそそる一文を入れました。

[![Image from Gyazo](https://i.gyazo.com/c26f6f97c70920898bc80fa758839c93.png)](https://gyazo.com/c26f6f97c70920898bc80fa758839c93)

検索するにはワード検索、セレクト検索、画像検索がありますが、ここでは画像検索についてご紹介します。

ヘッダーのドロップダウンから「画像検索」ボタンをクリックするとモーダルがでてきます。

そして検索したい虫の画像を入れて検索します。

[![Image from Gyazo](https://i.gyazo.com/d3628f434de250827f9d95813080f165.png)](https://gyazo.com/d3628f434de250827f9d95813080f165)

すると検索結果に何種類かの気持ち悪い系の虫が表示されます。

見た目がショッキングな虫がいらっしゃるので、モザイクをかけております。

[![Image from Gyazo](https://i.gyazo.com/04059db1826ae8c3ef3e2b36334816bd.png)](https://gyazo.com/04059db1826ae8c3ef3e2b36334816bd)

詳細ページにはその虫の「駆除方法」や「予防方法」、「実害」がどれくらいなのかを数値と文章で表示しております。

[![Image from Gyazo](https://i.gyazo.com/f447ef814b53f0c5e1b9bd9cb773b788.png)](https://gyazo.com/f447ef814b53f0c5e1b9bd9cb773b788)


### 使用技術
- Rails
- Ruby
- Rspec
- jQuery
- Bootstrap
- Nginx
- Puma
- AWS
  - VPC
  - EC2
     - Amazon Linux 2
  - Route53
  - RDS
     - MySQL
  - ALB
  - ACM
  - IAM
  - S3
  - Rekognition
  - Translate
- CircleCI
- Capistrano

### インフラ構成図
CircleCIでRspecとRubocopを自動でチェックしております。
Capistranoでデプロイを簡略化しております。

[![Image from Gyazo](https://i.gyazo.com/c6f5c2113c264ed5704732bd87863e59.png)](https://gyazo.com/c6f5c2113c264ed5704732bd87863e59)

### 苦労した点

#### 画像検索機能
こちらはAWSの機能である`Amazon Rekoginition`、`Amazon Translate`、`Amazon S3`を使っております。

AWSはほぼ触ったことがなかったのと、外部APIを使うのがはじめてだったので苦労しました。

また、rubyで`Amazon Rekoginition`のラベル検出機能を扱っている記事がなかったので、公式ドキュメントを読み漁ってなんとか実装しました。

#### インフラ
以前railsチュートリアルでherokuにデプロイしたことしかなかったので、インフラの知識はまったくありませんでした。

AWSについてはUdemyの講座を購入して基礎的なところを学んでから、Qiitaにある先輩方の記事を参考に導入していきました。

CircleCIについてはチュートリアルを大まかに読んでから、サンプルコードなどを参考に実際に動かしながら適宜修正していきました。

Capistranoでのデプロイではpumaのエラーが出たり、アプリ側のエラーが出たりしたことで、いろいろなログを見る癖がつきました。

#### 虫の画像集め

ニッチすぎる虫だとフリーの画像素材が見つからなかったりして大変でした笑

なのでイラストを使っている虫もありますが、今後有料の画像を購入したりして反映させていこうと思います。


### 最後に

はじめて個人サービスを開発してみて、楽しさと同時に学ぶべきことがまだまだたくさんあるなと痛感しました。

ただ自分のアイディアを形にして作り出せるエンジニアという職業にさらに惹かれました。

これからは就活も頑張っていきます！

最後まで読んでいただき、ありがとうございました！

ぜひアプリの方も使っていただけたら嬉しいです！

こちらのアプリもぜひ〜↓

https://qiita.com/kimorisan/items/35bac22bef7e06af9f60

