---
title: "Flutter × Firebase × ChatGPT で 暗記カードを自動で作るサービスを個人開発しました"
emoji: "📇"
type: "tech"
topics: ["flutter", "firebase", "googlecloud", "chatgpt", "個人開発"]
published: true
---

## 作ったサービス

PDFを送信するとその中の単語を暗記カードにしてくれるサービス（のデモ版）を作りました。
資格勉強をしている方や、専門用語を隙間時間で覚えたい方に向けたサービスです。

https://flash-pdf-card.web.app/

## どんなサービスか

PDFをアップロードすることでその内容を解析し、一問一答形式の暗記カードを作成します。
ログイン不要で使うことができます。
データの永続化はできませんが、直前の出力結果は見ることが可能です。

![alt](https://raw.githubusercontent.com/kou72/zenn/main/image/flashpdfcard.gif)

## なぜ作ったか

私がとある資格勉強を行っている時、教材がテキストしかなく、隙間時間にスマホで学習するのが難しい状況でした。
そのため暗記カードを自作していたのですが、カードを作るのにかなりの時間を取られてしまい、非常に効率の悪い勉強方法になってしまいました。

隙間時間の活用や繰り返しの単純暗記に暗記カードは有効ですが、それを作るのに時間がとられていては意味がありません。
そんな問題を解決するために今回のアプリを作りました。

想定しているユーザーは、**資格勉強をしている方や、専門用語を隙間時間で覚えたい方**です。
**通勤しながらスマホで暗記カードを作って勉強する**、という使い方をイメージしています。

## 「作り過ぎない」を意識

今回のアプリは、まだ需要があるのかも分からないため **作り過ぎない** ことを意識しました。
本来はモバイルアプリとしてストアに出したいところですが、まずはWeb版だけでリリース。
また、 PDFから暗記カードが作られるというコアな体験のみを実装し、他の機能は全て削りました。

これはMVPをまずリリースせよ、という考え方に従っています。
MVPについて気になる方は、こちらの記事が面白いので読んでみてください。

https://www.ankr.design/designtips/making-sense-of-mvp

### 削った機能

- ログイン機能
- 暗記カードの永続保存
- カードの編集機能
- お気に入りやグループ分けなどオプション機能

などなど

## 使った技術

使用した技術は以下の通りです。

- [Flutter on the Web](https://flutter.dev/multi-platform/web)：フロント
- [Firebase](https://firebase.google.com/?hl=ja)：バックエンド全般
  - Firebase Hosting
  - Cloud Functions for Firebase
  - Cloud Storage for Firebase
- [Cloud Vision API](https://cloud.google.com/vision?hl=ja)：PDFからテキストを抽出
- [ChatGPT API](https://openai.com/chatgpt)：問題文の作成

### Flutter on the Web

将来的にモバイルアプリ化しやすく、かつデモ版はWebでリリースしたかったため、Flutter on the Webを採用しました。
このような要件がある場合はFlutter on the Webが有力な選択肢になると思います。

ただしブラウザとモバイルではローカルリソースの扱いなどがかなり異なるため、このまま何も手を加えずにモバイル化できるわけではないので注意です。

#### 注意点）ファイルのパスにアクセスできない

以下の記事と同じ、file_pickerのpathが使えない問題、またパスで指定したデータをうまく送れない問題に遭遇しました。

https://zenn.dev/enoiu/scraps/eef936965c77ab

エラー内容はこちら。

```sh
Error: Unsupported operation: MultipartFile is only supported where dart:io is available.
```

そのまま読むと`dart:io`そのものが Web 版だと使えないような気がしてしまいますが、原因はブラウザが任意のファイルパスの読み込みをサポートしていないことです。

パスではなくbyteでファイルのデータを与えることで解決することができます。

https://github.com/flutter/flutter/issues/98208

> ブラウザは任意のファイルパスの読み込みをサポートしていないため、ブラウザで動作しないのは特にMultipartFile.fromPathである。
> ブラウザでサポートされているメソッドを使用してファイルを選択し、バイトまたは文字列として利用できる場合は、MultipartFile.fromStringまたはMultipartFile.fromBytesを使用できます。
> (記事から引用し翻訳)

### Firebase

使い慣れていることからバックエンドはマルっと Firebase にすることとしました。
またFirebaseの裏側はGCPで出来ているため、同じプロジェクトにCloud Vision APIも一緒に作成できたり、ログもGCPのログエクスプローラから一括で見れたりと、管理面で楽に開発を進めることができました。

#### Firebase Hosting

`web.app`のドメイン取得やHTTPS化だけでなく使用状況のダッシュボードも付いてきます。
これもFirebaseに決めた理由の一つです。

#### Cloud Functions for Firebase

セキュリティの観点から**クライアントに認証情報を持たせない**ため、ストレージへや各APIへのアクセスは全てCloud Functionsを経由して行う設計にしました。

こうすることでFlutter側にFirebase関連のライブラリを入れる必要もなくなり、フロントエンドを切り離すことも容易になったため、良かったのではないかなと思っています。

![alt](https://raw.githubusercontent.com/kou72/zenn/main/image/flutter-request-pattern.png)

ただこの設計にしたため、ストレージにファイルをアップロードする実装で幾つかハマってしまいました。
Functionを経由したファイルアップロードの苦労話はこちらに書いているので、興味があれば読んでみてください。

https://zenn.dev/kou7273/articles/multipart-file-upload-with-flutter-to-firebase

#### Cloud Storage for Firebase

Cloud Vision API の仕様上 Cloud Storage 上のファイルを指定する必要があり導入しました。
仕方なし。

### Cloud Vision API

一度使ったことがあったこと、GCPで管理できること、テキスト抽出の精度が高そうであることから採用しました。
精度については非常に良く、料金も良心的で満足しています。

ただし以下2点の仕様があまり良くなく、想定よりも実装サイズが大きくなってしまいました。

#### 想定外の仕様1) Cloud Storage のファイルを指定する必要がある

これはちょっと残念な仕様。ストレージレスのサービスにしたかったのですが断念しました。

https://cloud.google.com/vision/docs/pdf?hl=ja

> Vision API では、Cloud Storage に保存されている PDF ファイルと TIFF ファイルのテキストを検出して転写できます。
> (ドキュメントから引用)

#### 想定外の仕様2) 出力の json ファイルが Cloud Storage 上に配置される

これはかなり残念というか困惑した仕様。
Vision API を叩く際に必ず出力先となる Cloud Storage のパスを指定する必要があり、**テキスト抽出された結果はそのパスに json ファイルとして出力されます**。
そのため、出力をのjsonの内容を取得しパースするという処理が必要になります。

またjsonのファイルは `output-1-to-6.json` のようなファイル名で固定されていて、**あらかじめ指定することができません**。
元の**PDFのページ数によってファイル名も変わる**ので、jsonの内容を取得するにしても一々ファイル名も確認しなければなりません。

このあたりの実装でかなり時間を使ってしまった気がします。

### ChatGPT API

テキストから問題文と回答のセットを作成するために使いました。
私のプロンプト力だと GPT-3.5 で対応させることが難しく、ちょっと贅沢ですが GPT-4 に投げています。

system プロンプトは以下のようにしています。

```sh
テキスト中暗記するべき専門用語を抽出し、それらが回答となる問題文を作成してください。
問題文は文脈関係なく回答可能なように、前提条件や背景込みで出題すること。
回答は重複させないこと。フォーマットはjson形式とすること。
```

これできちんとjson形式に整形されて 問題文と回答のセットが返ってきます。
すごいですね。

## 最後に

今回作ったアプリはまだデモ版になります。
ここから本実装に向けて、たくさんのフィードバックをいただけると嬉しいです。
興味のある方は使ってみていただき、改善点や追加したい機能などがあれば是非とも教えてください！

https://flash-pdf-card.web.app/

また、今回の記事がどなたかの開発の助けになれば幸いです。
