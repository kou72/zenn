---
title: "Flutter から Firebase Functions へ multipart/form-data でファイルを送信する"
emoji: "📮"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "firebase", "googlecloud", "html", "個人開発"]
published: true
---

# 背景

FlutterとFirebaseを用いたアプリ開発は現在ではよくある構成かと思いますが、私も同じ構成でアプリを開発しています。
アプリの機能としてユーザーからファイルのアップロードを受け付ける必要があり、その際にFirebase Storageを利用することにしました。

しかし、FlutterからFirebase Storageへ直接ファイルをアップロードしようとすると、クライアントに認証情報を埋め込む必要があります。
クライアントへの認証情報の埋め込みはセキュリティ上のリスクがあるため、Firebase Functionsを経由してファイルのアップロードを実施することにしました。

![](https://raw.githubusercontent.com/kou72/zenn/main/image/flutter-request-pattern.png)

# 開発環境

- Flutter
  - ver 3.10.6
  - Flutter for Web のみ利用
- Firebase Functions
  - node 18
  - typescript で実装

# ファイルアップロード方法について

RESTful WebAPI でファイルをアップロードする場合、主に「multipart/form-data」と「Base64 エンコード」の2つの方法があるようです。
この点については以下の記事が参考になります。

https://qiita.com/mserizawa/items/7f1b9e5077fd3a9d336b

> ### multipart/form-data
>
> これは HTTP でのファイル送信の王道といえます。RFC2388 でも定義されていて、ほとんどの Web アプリケーションフレームワークでは、この手法で送られてきたファイルをデコードする仕組みを持っています。
>
> ### Base64
>
> ファイルのデータを Base64 エンコードして文字列化したものを、JSON につめて送信する手法です。(略)JSON ベースの API と親和性が取れるメリットがありますが、データ容量がかさむのと、エンコード／デコードのコストがかかるというデメリットがあります。

([記事](https://qiita.com/mserizawa/items/7f1b9e5077fd3a9d336b)から引用)

基本的にはmultipart/form-dataでファイルをアップロードするのが良いように思えたため、今回はこの方法を採用することにしました。

また、Cloud Storageがストリーミングアップロードに対応しているため、このアーキテクチャとの相性も良いと考えました。

# multipart/form-data とは

multipart/form-dataは、複数のデータをbody部分に含めて送信します。
データの区切りには、Content-Typeで指定されたboundaryという文字列が使用されます。

**Headers**

```http
Host: example.org
Content-Type: multipart/form-data; boundary=----------XnJLe9ZIbbGUYtzPQJ16u1
Cookie:
Origin:
```

**Body**

```http
------------XnJLe9ZIbbGUYtzPQJ16u1
Content-Disposition: form-data; name="title"

title1
------------XnJLe9ZIbbGUYtzPQJ16u1
Content-Disposition: form-data; name="description"

description1
------------XnJLe9ZIbbGUYtzPQJ16u1
Content-Disposition: form-data; name="stream_id"

1019900359
------------XnJLe9ZIbbGUYtzPQJ16u1
Content-Disposition: form-data; name="item_images[]"; filename="sample.png"
Content-Type: image/png
Content-Length: 4323

{ Put binary contents that you want to upload }
```

（[tab API](http://tonchidot.github.io/tab-api-docs/api/item/create_new_item.html) の送信サンプルより引用）

受け取り側は、boundaryをもとにデータを分割し、中身を取り出す必要があります。
そのため、データを正確に取得するためには、送信側と受信側の双方でmultipart/form-data用のライブラリの利用が必須です。

# Flutter で multipart/form-data を活用したファイルアップロード

## 利用パッケージ

FlutterでRESTful WebAPIを含めhttpを取り扱うときには、[http](https://pub.dev/packages/http) か [dio](https://pub.dev/packages/dio) を選択して利用することとなります。

以下の記事で二つのパッケージの比較が紹介されています。
https://medium.com/@vikranthsalian/flutter-dio-vs-http-1dc1d4f95fda

> ### HTTP
>
> - メリット
>   - Dartの開発者によって提供されている
>   - クライアントとサーバーの間の標準的なやりとりのためのもの
> - デメリット
>   - 基本的な機能しか提供していない。追加の機能は独自に実装する必要がある
>   - エラーハンドリングのための追加の関数を書く必要がある
>
> ### DIO
>
> - メリット
>   - 多くの機能を提供している。基本的なネットワーク処理に加えて、dioはインターセプター、ログ、キャッシュなどの追加機能も提供している
>   - クライアントをできるだけ迅速に実装することができる。コードの量を減らし、開発プロセスを加速する追加の抽象化を持っている。
> - デメリット
>   - 信頼性。提供される機能の数が多いため、パッケージにバグがある可能性が高まる。

([記事](https://medium.com/@vikranthsalian/flutter-dio-vs-http-1dc1d4f95fda)からの引用をChat-GPTにより和訳)

今回の実装では、初めにhttpを使用し、後に必要があればdioに切り替えることにしました。
この記事ではhttpのみで十分な実装ができました。

## 実装方法

Flutterでの実装例は以下の通りです。

```dart
import 'dart:typed_data';
import 'package:flutter/material.dart';
import 'package:file_picker/file_picker.dart';
import 'package:http/http.dart' as http;

class Screen extends StatefulWidget {
  const Screen({Key? key}) : super(key: key);
  @override
  ScreenState createState() => ScreenState();
}

class ScreenState extends State<Screen> {
  String? _pickedFileName = '';
  Uint8List? _pickedFileBytes;

  Future<void> _pickPDF() async {
    FilePickerResult? result = await FilePicker.platform.pickFiles(
      type: FileType.custom,
      allowedExtensions: ['pdf'],
    );

    if (result == null) return;
    PlatformFile file = result.files.first;
    setState(() {
      _pickedFileName = file.name;
      _pickedFileBytes = file.bytes;
    });
  }

  Future<void> _fileupload() async {
    final url = Uri.https('fileupload.a.run.app');
    final req = http.MultipartRequest('POST', url);
    req.files.add(
      http.MultipartFile.fromBytes(
        'file',
        _pickedFileBytes!,
        filename: _pickedFileName!,
      ),
    );
    req.send();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('PDF Upload'),
      ),
      body: Column(
        children: <Widget>[
          ElevatedButton(
            onPressed: _pickPDF,
            child: const Text('PDFを選択する'),
          ),
          ElevatedButton(
            onPressed: _fileupload,
            child: const Text('ファイルをアップロードする'),
          ),
        ],
      ),
    );
  }
}
```

この中で、`_fileupload` の部分が `http` を使用して、multipart/form-data形式でデータを送信する処理です。

```dart
final req = http.MultipartRequest('POST', url);
req.files.add(
  http.MultipartFile.fromBytes(
    'file',
    _pickedFileBytes!,
    filename: _pickedFileName!,
  ),
);
req.send();
```

このコードを、前述のmultipart/form-dataのbodyの例と比較すると、データの組み立て方がよくわかるかと思います。
`http.MultipartRequest` で新しいリクエストを作成し、その後 `req.files.add` でファイルデータを追加しています。

各パラメータの詳細は以下の通りです。

- **(第1引数): 'file'**
  - これはデータのname属性を指定する部分です。
- **(第2引数): \_pickedFileBytes!**
  - 実際のファイルデータ（バイナリデータ）を指定する部分です。
- **filename: \_pickedFileName!**
  - アップロードするファイルの名前を指定する部分です。
  - 詳細は[公式ドキュメント](https://pub.dev/documentation/http/latest/http/MultipartFile/MultipartFile.fromBytes.html)を参照してください。

## 注意

開発中、以下のエラーが発生しました。

```bash
Error: Unsupported operation: MultipartFile is only
supported where dart:io is available.
```

原因は `http.MultipartFile.fromPath` メソッドを使用していたためで、Flutter for Webではこのメソッドは利用できないようです。

https://github.com/flutter/flutter/issues/98208

> ブラウザは任意のファイルパスの読み込みをサポートしていないため、ブラウザで動作しないのは特にMutlipartFile.fromPathである。
> ブラウザでサポートされているメソッドを使用してファイルを選択し、バイトまたは文字列として利用できる場合は、MultipartFile.fromStringまたはMultipartFile.fromBytesを使用できます。

([記事](https://github.com/flutter/flutter/issues/98208)からの引用をChat-GPTにより和訳)

ここでは代わりに `http.MultipartFile.fromBytes` を利用することとしました。

# Firebase Functionsでのmultipart/form-dataの取り扱い

## 必要なパッケージ

Firebase Functions で multipart/form-data を取り扱う際 [busboy](https://www.npmjs.com/package/busboy) というパッケージを使用します。
他にも類似するパッケージとして [Formidable](https://www.npmjs.com/package/formidable) 、[Multer](https://www.npmjs.com/package/multer) 、[Multiparty](https://www.npmjs.com/package/multiparty) などがありますが、これらは Cloud Functions では現在サポートされていないようです。
（[公式ドキュメント](https://cloud.google.com/functions/docs/samples/functions-http-form-data?hl=ja)では busboy が使われている & 報告記事多数）

こちらの記事が分かりやすいですが、中間ファイルの利用するものは利用できないようです。

https://bytearcher.com/articles/formidable-vs-busboy-vs-multer-vs-multiparty/

![](https://bytearcher.com/articles/formidable-vs-busboy-vs-multer-vs-multiparty/flowchart-nobg.png)
([記事](https://bytearcher.com/articles/formidable-vs-busboy-vs-multer-vs-multiparty/)の画像を引用)

## Firebase Strage へのアップロード

busboy はファイルをストリーム形式で受け取ることができます。

```js
const bb = busboy({ headers: req.headers });
bb.on("file", (name, file, info) => {
  const saveTo = path.join(os.tmpdir(), `busboy-upload-${random()}`);
  file.pipe(fs.createWriteStream(saveTo));
});
```

(ファイル受信の[サンプルコード](https://github.com/mscdex/busboy#examples)から引用)

https://github.com/mscdex/busboy#examples

また、Firebase Storageも同様のストリーム形式でファイルをアップロードすることができます。

```js
const stream = require("stream");
const storage = new Storage();
const myBucket = storage.bucket(bucketName);
const file = myBucket.file(destFileName);

const passthroughStream = new stream.PassThrough();
passthroughStream.write(contents);
passthroughStream.end();

async function streamFileUpload() {
  passthroughStream.pipe(file.createWriteStream()).on("finish", () => {
    // The file upload is complete
  });
  console.log(`${destFileName} uploaded to ${bucketName}`);
}
```

(ストリーミングアップロードの[サンプルコード](https://cloud.google.com/storage/docs/streaming-uploads?hl=ja#storage-stream-upload-object-nodejs)から引用)

https://cloud.google.com/storage/docs/streaming-uploads?hl=ja#storage-stream-upload-object-nodejs

これらを組み合わせて、busboyで受け取ったファイルを、Firebase Storageにそのままストリームでアップロードすることが可能です。
一時ファイルの使用や余計なメモリの消費を避けつつ、効率的にファイルをアップロードすることができます。

Firebase のドキュメントでも言及されているので、基本的にはこの実装が良さそうです。

https://firebase.google.com/docs/functions/tips?hl=ja#always_delete_temporary_files

> 一時ディレクトリ以外への書き込みはしないでください。また、プラットフォームや OS に依存しない方法でファイルパスを構築してください。
>
> パイプラインを使用して、サイズの大きいファイルを処理する際のメモリ要件を減らすことができます。たとえば、Cloud Storage でファイルを処理するために、読み取りストリームを作成し、これをストリームベースのプロセスに渡してから、出力ストリームを Cloud Storage に直接書き込むことができます。

## 実装方法

Firebase Functionsでの実装例を以下に示します。

```js
import { onRequest } from "firebase-functions/v2/https";
import * as admin from "firebase-admin";
import * as busboy from "busboy";
import { getDate } from "./utils"; // yyyyMMddHHmmss形式で日付を取得する関数

admin.initializeApp();

const storage = admin.storage();
const bucket = storage.bucket();

export const fileupload = onRequest({ timeoutSeconds: 300, cors: true }, (req, res) => {
  const bb = busboy({ headers: req.headers });
  bb.on("file", (name, stream, info) => {
    const date = getDate();
    const distPath = bucket.file(`upload/${date}-${info.filename}`);
    stream.pipe(distPath.createWriteStream()).on("finish", () => {
      res.status(200).send("File processed.");
    });
  });
  bb.end(req.rawBody);
});
```

この実装では、`bb.on("file")` 内でファイルデータを取得し、バイナリデータが入っている `stream` と `info.filename` を利用してFirebase Storageにアップロードしています。

アップロード処理には `stream.pipe(distPath.createWriteStream())` を使用することで、中間ファイルやメモリを挟まず、効率的にファイルを保存することができます。

# まとめ

- FlutterとFirebaseを組み合わせたアプリ開発において、クライアントからFirebase Storageへ直接アップロードするとセキュリティリスクがあるため、Firebase Functionsを経由してアップロードを行う方法を検討しました。
- ファイルアップロード方法としては「multipart/form-data」と「Base64 エンコード」の2つが主流で、今回は前者を採用しました。
- Flutterではhttpパッケージを使用してmultipart/form-data形式でデータを送信しました。
- Firebase Functionsではbusboyパッケージを使用してmultipart/form-dataを取り扱い、受け取ったファイルをFirebase Storageにストリームでアップロードしました。

FlutterとFirebaseを組み合わせたアプリ開発を行っている方の、ファイルアップロード実装の参考になれば幸いです。
