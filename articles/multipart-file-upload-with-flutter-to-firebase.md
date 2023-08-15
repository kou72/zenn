---
title: "Flutter から Firebase Functions へ multipart/form-data でファイルを送信する"
emoji: "📮"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "firebase", "googlecloud", "html", "個人開発"]
published: false
---

# 1

FlutterとFirebaseを使ったアプリ開発を行っている
FlutterからFirebase Strageへファイルをアップロードする必要があった。
Flutter から直接 Firebase Strage へファイルをアップロードするためにはクライアントに認証情報を埋め込む必要がある。
しかし、クライアントに認証情報を埋め込むのはセキュリティ的に良くないので、Firebase Functions を経由してファイルをアップロードすることにした。

# 2

RESTful WebAPI でファイルをアップロードする場合、主に「multipart/form-data」と「Base64 エンコード」の2つの方法があるようだ。
この点については以下の記事が参考になる。

https://qiita.com/mserizawa/items/7f1b9e5077fd3a9d336b

> ### multipart/form-data
>
> これは HTTP でのファイル送信の王道といえます。RFC2388 でも定義されていて、ほとんどの Web アプリケーションフレームワークでは、この手法で送られてきたファイルをデコードする仕組みを持っています。
>
> ### Base64
>
> ファイルのデータを Base64 エンコードして文字列化したものを、JSON につめて送信する手法です。(略)JSON ベースの API と親和性が取れるメリットがありますが、データ容量がかさむのと、エンコード／デコードのコストがかかるというデメリットがあります。

([記事](https://qiita.com/mserizawa/items/7f1b9e5077fd3a9d336b)から引用)

基本的にはmultipart/form-dataでファイルをアップロードするのが良いように思える。今回はこの方法を採用することにした。

また後述するが、Cloud Strageがストリーミングアップロードに対応しているため、アーキテクチャとも相性が良い。

# multipart/form-data

multipart/form-data では複数のデータをbodyに含めて送信する。
データの区切りにはContent-Typeで指定されたboundaryと呼ばれる文字列を用いられる。

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

また、受領側はboundaryを元にデータを分割して中身を取り出す必要がある。

そのため適当にボディを開いてもデータを取り出すことはできず、送信側、受信側ともにmultipart/form-data用のライブラリを利用する必要がある。

# Flutter から multipart/form-data でファイルをアップロードする

## 利用パッケージ

FlutterでRESTful WebAPIを含めhttpを取り扱うときには、[http](https://pub.dev/packages/http) か [dio](https://pub.dev/packages/dio) を選択して利用することとなる。

こちらの記事が比較の参考になる。
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

今回はhttpで実装後、必要であればdioに切り替える方針とした。
本記事の範囲ではhttpで十分に実装できた。

## 実装

Flutter側の実装は以下のようになる。

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

`_fileupload` 中の以下の部分がhttpを使ってmultipart/form-dataで送信するデータを組み立てている処理になる。

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

上のmultipart/form-dataのbodyの例と見比べるとイメージしやすくなると思う。
`http.MultipartRequest` で空のリクエストを作成後、 `req.files.add` でファイルのデータを追加している。

各パラメータの意味は以下の通り。

- **(第1引数): 'file'**
  - データのnameを指定している
- **(第2引数): \_pickedFileBytes!**
  - バイナリデータをセットしている
- **filename: \_pickedFileName!**
  - オプションで設定できるファイルの名前

詳細は[公式ドキュメント](https://pub.dev/documentation/http/latest/http/MultipartFile/MultipartFile.fromBytes.html)を参照。

## 注意

開発中以下のエラーが発生した。

```bash
Error: Unsupported operation: MultipartFile is only
supported where dart:io is available.
```

これは `http.MultipartFile.fromPath` メソッドを利用していたことで発生したものであった。
Flutter for Web では `http.MultipartFile.fromPath` が使えないようである。

https://github.com/flutter/flutter/issues/98208

> ブラウザは任意のファイルパスの読み込みをサポートしていないため、ブラウザで動作しないのは特にMutlipartFile.fromPathである。
> ブラウザでサポートされているメソッドを使用してファイルを選択し、バイトまたは文字列として利用できる場合は、MultipartFile.fromStringまたはMultipartFile.fromBytesを使用できます。

([記事](https://github.com/flutter/flutter/issues/98208)からの引用をChat-GPTにより和訳)

ここでは `http.MultipartFile.fromBytes` を利用することで解決した。

# Firebase Functions で multipart/form-data を受け取る

## 利用パッケージ

Firebase Functions で multipart/form-data を受け取るときには [busboy](https://www.npmjs.com/package/busboy) を利用する。
他に類似のライブラリとして [Formidable](https://www.npmjs.com/package/formidable) 、[Multer](https://www.npmjs.com/package/multer) 、[Multiparty](https://www.npmjs.com/package/multiparty) があるが、非サポートのようです。
（[公式ドキュメント](https://cloud.google.com/functions/docs/samples/functions-http-form-data?hl=ja)では busboy が使われている & 報告記事多数）

こちらの記事が分かりやすいですが、中間ファイルの利用するかどうかが関係してそうです。

https://bytearcher.com/articles/formidable-vs-busboy-vs-multer-vs-multiparty/

![](https://bytearcher.com/articles/formidable-vs-busboy-vs-multer-vs-multiparty/flowchart-nobg.png)
([記事](https://bytearcher.com/articles/formidable-vs-busboy-vs-multer-vs-multiparty/)の画像を引用)

## Firebase Strage へのアップロード

busboyはstreamでファイルを受けとることが可能です。

```js
const bb = busboy({ headers: req.headers });
bb.on("file", (name, file, info) => {
  const saveTo = path.join(os.tmpdir(), `busboy-upload-${random()}`);
  file.pipe(fs.createWriteStream(saveTo));
});
```

(ファイル受信の[サンプルコード](https://github.com/mscdex/busboy#examples)から引用)

https://github.com/mscdex/busboy#examples

またFirebase Strageにはstreamでファイルをアップロードすることが可能です。

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

そのため、busboyで受け取ったファイルをstreamでそのままFirebase Strageにアップロードすることが可能です。
こうすることで中間ファイルを利用することなく、またメモリ消費することなくファイルアップロードが可能です。

Firebase のドキュメントでも言及されているので、基本的にはこの実装が良さそうです。

https://firebase.google.com/docs/functions/tips?hl=ja#always_delete_temporary_files

> 一時ディレクトリ以外への書き込みはしないでください。また、プラットフォームや OS に依存しない方法でファイルパスを構築してください。
>
> パイプラインを使用して、サイズの大きいファイルを処理する際のメモリ要件を減らすことができます。たとえば、Cloud Storage でファイルを処理するために、読み取りストリームを作成し、これをストリームベースのプロセスに渡してから、出力ストリームを Cloud Storage に直接書き込むことができます。

## 実装

Firebase Functions 側の実装は以下のようになる。

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

`bb.on("file")` でnameがfileのデータに対する処理を記述している。
`stream`にはファイルのバイナリデータ、`info.filename`にファイル名が入っているため、これらを利用してFirebase Strageにアップロードしている。

アップロードには`stream.pipe(distPath.createWriteStream())`を利用することで、中間ファイルやメモリを介さずにアップロードが可能になる。

# まとめ

- RESTful WebAPI でファイルアップロードを行う場合、multipart/form-data を利用が適している。
- Flutter でファイルアップロードを行う場合、[MultipartRequest](https://api.flutter.dev/flutter/dart-io/MultipartRequest-class.html) を利用する。
- Firebase Functions でファイルアップロードを行う場合、[busboy](https://www.npmjs.com/package/busboy) を利用する。
- Firebase Strage へのファイルアップロードは、[stream](https://nodejs.org/api/stream.html) を利用することで中間ファイルやメモリを介さずに行うことが可能である。
