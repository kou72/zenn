---
title: "FlutterでPDFのページ数を確認したいだけ"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "個人開発"]
published: true
---

# PDFのページ数を確認したい

Flutterでアプリ開発中、読み込んだPDFのページ数を確認したいだけなのにえらく時間がかかってしまったのでメモ。

# 開発環境

- Flutter
  - ver 3.10.6
  - Flutter for Web を利用

# ライブラリがたくさんある

PDF系のライブラリがなぜかたくさんあります。
結論としては [pdfx](https://pub.dev/packages/pdfx) でやりたいことができました。

- **使ってみたライブラリ**

  - [pdf](https://pub.dev/packages/pdf)
  - [printing](https://pub.dev/packages/printing)
  - [pdf_render](https://pub.dev/packages/pdf_render)
  - [native_pdf_renderer](https://pub.dev/packages/native_pdf_renderer) ※pdfxにリプレース
  - [native_pdf_view](https://pub.dev/packages/native_pdf_view) ※pdfxにリプレース
  - [**pdfx**](https://pub.dev/packages/pdfx)

- **使わなかったけど調べて出てきたライブラリ**
  - [advance_pdf_viewer](https://pub.dev/packages/advance_pdf_viewer)
  - [syncfusion_flutter_pdfviewer](https://pub.dev/packages/syncfusion_flutter_pdfviewer)
  - [flutter_full_pdf_viewer](https://pub.dev/packages/flutter_full_pdf_viewer)
  - [flutter_pdfview](https://pub.dev/packages/flutter_pdfview)

# ドキュメントが分かりにくい

私の知識不足もありますが、どのライブラリもドキュメントが分かりにくいと思います。
利用例をもうちょっと増やしてほしい。。。

[pdfのリファレンス](https://pub.dev/documentation/pdf/latest/pdf/pdf-library.html) など、ちょっと分かりにくい。
[**pdfx**のリファレンス](https://pub.dev/documentation/pdfx/latest/) は分かりやすいです。

# Flutter for Webでうまくいかないライブラリがある

ブラウザは**任意のファイルパスの読み込みをサポートしていない**らしく、ファイルのパスを直接指定するタイプのライブラリは避けることにしました。

pdfライブラリの文脈とは違いますが、こちらのコメントを参考にしてます。

https://github.com/flutter/flutter/issues/98208

> ブラウザは任意のファイルパスの読み込みをサポートしていないため、ブラウザで動作しないのは特にMutlipartFile.fromPathである。
> ブラウザでサポートされているメソッドを使用してファイルを選択し、バイトまたは文字列として利用できる場合は、MultipartFile.fromStringまたはMultipartFile.fromBytesを使用できます。

([記事](https://github.com/flutter/flutter/issues/98208)からの引用をChat-GPTにより和訳)

# pdfxを使ってみる

[**pdfx**](https://pub.dev/packages/pdfx)で進めますがいくつか注意があります。

## Webで利用する場合は別途コマンドの実行が必要です。

Flutter for webで開発している時は `flutter pub run pdfx:install_web` の実行が必要なので忘れないようにしましょう。

> ### Getting Started
>
> In your flutter project add the dependency:
>
> ```
> flutter pub add pdfx
> ```
>
> For web run tool for automatically add pdfjs library (CDN) in index.html:
>
> ```
> flutter pub run pdfx:install_web
> ```
>
> For windows run tool automatically add override for pdfium version property in CMakeLists.txt file:
>
> ```
> flutter pub run pdfx:install_windows
> ```

([pdfx](https://pub.dev/packages/pdfx)のドキュメントから引用)

## pdfController ではうまくいかない

ドキュメントには `pdfController.pagesCount` で総ページ数を取得するよう書かれていますが、うまくいきませんでした。（なぜ）

代わりに古いライブラリの書き方である `docFromData.pagesCount` でうまく取得することができました。

## pdfxを使ったPDFのページ数取得のサンプルコード

以下のコードでPDFの総ページ数を取得することができます。

```dart
PdfDocument docFromData = await PdfDocument.openData(file.bytes);
final pdfPagesCount = docFromData.pagesCount
```

私のアプリでは `_pickPDF()` というPDFを選択してアップロードする関数の中でこんな感じで使っております。

```dart
Future<void> _pickPDF() async {
  FilePickerResult? result = await FilePicker.platform.pickFiles(
    type: FileType.custom,
    allowedExtensions: ['pdf'],
  );

  if (result == null || result.files.isEmpty) return;
  PlatformFile file = result.files.first;
  PdfDocument docFromData = await PdfDocument.openData(file.bytes!);

  if (docFromData.pagesCount >= _maxPageCount) {
    if (!mounted) return;
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(
          content: Text('$_maxPageCount ページ以下のファイルを選択してください。'),
          backgroundColor: Colors.blueGrey),
    );
    return;
  }
  setState(() {
    _pickedFileName = file.name;
    _pickedFileBytes = file.bytes;
  });
}
```

# まとめ

- PDF系のライブラリあり過ぎ & ちょっと使いにくい
- [**pdfx**](https://pub.dev/packages/pdfx) のドキュメントが分かりやすい！
- [**pdfx**](https://pub.dev/packages/pdfx) を使おう！

誰かの参考になれば幸いです。
