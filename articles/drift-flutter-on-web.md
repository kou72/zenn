---
title: "Drift を Flutter Web で使う"
emoji: "🚗"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "個人開発"]
published: true
---

## Flutter WebでローカルDBを使いたい

Flutterでアプリ開発している中で、データの永続化にローカルDBを導入する必要が出てきました。
Flutter用のローカルDBライブラリはかなりの数ありますが、今回はWebでも利用可能な Drift を使ってみることにしました。
Flutter Web で Drift を利用しようとすると、ドキュメントだけでは実装が難解な部分があるのでこの記事で解説しようと思います。

## Flutter Web の制限

Flutter はモバイルアプリ開発のためのフレームワークですが、Webでも利用することができます。
私はモバイルアプリとしてリリースする前にデモ版をWebで公開するようにしていて、Flutter Web はそのために利用しています。

ただし**ブラウザが任意のファイルパスの読み込みをサポートしていない**ため`dart:io`やローカルストレージを素直に利用することができず、関連するライブラリの選定や利用方法には注意が必要です。

## Drift とは

[Drift](https://drift.simonbinder.eu) は Web にも対応している Flutter 用のローカルDBライブラリです。
SQLite をラップして抽象化したもので、備えている ORM を使ったDB操作が可能です。
また [Flutter Favorite program](https://docs.flutter.dev/packages-and-plugins/favorites) にも掲載されているため、品質も信用できそうです。

:::message
[**Flutter Favorite program**](https://docs.flutter.dev/packages-and-plugins/favorites) とは、品質が高く信頼できると評価された Flutter パッケージとプラグインを Flutter 公式が推奨する取り組みです。推奨されているライブラリは総合的なパッケージスコアやライセンス、機能の完全性などの厳格な基準を満たしています。
:::

## Drift を Flutter Web で使う

Drift を Flutter Web で使うには[通常のセットアップ手順](https://drift.simonbinder.eu/docs/getting-started/)のうち一部の手順を[こちらのページ](https://drift.simonbinder.eu/web/)に書いてあるもので差し替えて進める必要があります。

### ライブラリの導入

まずは`pubspec.yaml`にライブラリを追加してインストールします。

```yaml:pubspec.yaml
dependencies:
  drift: ^2.13.0
dev_dependencies:
  drift_dev: ^2.13.0
  build_runner: ^2.4.6
```

`drift`: ライブラリ本体となるコアパッケージ
`drift_dev`: リコードを生成に必要な開発用ツール
`build_runner`: コード生成のための一般的なツール

`sqlite3_flutter_libs`, `path_provider`, `path` のインストールは不要です。

次に

- `sqlite3.wasm`を[sqlite3のリリースページ](https://github.com/simolus3/sqlite3.dart/releases)から
- `drift_worker.js`を[driftのリリースページ](https://github.com/simolus3/drift/releases)から

ダウンロードして`web`ディレクトリに配置します。
`web/` がこのようになればOK。

```md
web/
├── favicon.png
├── index.html
├── manifest.json
├── drift_worker.js
└── sqlite3.wasm
```

### データベースクラスの作成

`lib`ディレクトリの適当な場所にデータベースクラスを作成します。
今回は`drift`ディレクトリ配下に作成することにします。
`Decks`とは私が実際にアプリで利用しているテーブルの名前です。

```dart:lib/drift/decks_database.dart
import 'package:drift/drift.dart';
part 'decks_database.g.dart';

class Decks extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get title => text()();
}

@DriftDatabase(tables: [Decks])
class DecksDatabase extends _$DecksDatabase {}
```

build_runnerのコマンドで`database.g.dart`を自動生成します。

```bash
flutter pub run build_runner build --delete-conflicting-outputs
```

:::message
**--delete-conflicting-outputs** オプションをつけることで競合しているファイルを一旦削除して再作成してくれます。
:::

### Webデータベースと接続する

`NativeDatabase` の代わりに `WasmDatabase` を使って作成したクラスをWebデータベースと接続します。
`WasmDatabase` は `package:drift/wasm.dart` から呼び出します。

```diff:lib/drift/decks_database.dart
import 'package:drift/drift.dart';
+ import 'package:drift/wasm.dart';
part 'decks_database.g.dart';

class Decks extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get title => text()();
}

@DriftDatabase(tables: [Decks])
class DecksDatabase extends _$DecksDatabase {
+  DecksDatabase._(QueryExecutor e) : super(e);
+  factory DecksDatabase() => DecksDatabase._(connectOnWeb());
+  @override
+  int get schemaVersion => 1;
}

+ DatabaseConnection connectOnWeb() {
+   return DatabaseConnection.delayed(Future(() async {
+     final result = await WasmDatabase.open(
+       databaseName: 'decks_db',
+       sqlite3Uri: Uri.parse('sqlite3.wasm'),
+       driftWorkerUri: Uri.parse('drift_worker.js'),
+     );
+
+     if (result.missingFeatures.isNotEmpty) {
+       print('Using ${result.chosenImplementation} due to missing browser '
+           'features: ${result.missingFeatures}');
+     }
+
+     return result.resolvedExecutor;
+   }));
+ }
```

基本的にはドキュメントの `connectOnWeb` をそのまま使っています。
セットアップはこれで完了です。

### メソッドを作成する

ここからはモバイルと同様です。
DB操作用のお好みのメソッドを作成します。

```dart:lib/drift/decks_database.dart
...省略

@DriftDatabase(tables: [Decks])
class DecksDatabase extends _$DecksDatabase {
  DecksDatabase._(QueryExecutor e) : super(e);
  factory DecksDatabase() => DecksDatabase._(connectOnWeb());
  @override
  int get schemaVersion => 1;

  // メソッドを設定
  Future insertDeck(String title) {
    return into(decks).insert(DecksCompanion.insert(title: title));
  }

  Future deleteDeck(int id) {
    return (delete(decks)..where((deck) => deck.id.equals(id))).go();
  }

  Future updateDeck(int id, String title) {
    return (update(decks)..where((deck) => deck.id.equals(id)))
        .write(DecksCompanion(title: Value(title)));
  }
}

...省略
```

### DBを呼び出す

DBを呼び出して使います。

```dart:lib/main.dart
import 'package:flutter/material.dart';
import '../drift/decks_database.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  final db = DecksDatabase();

  await db.insertDeck('test');
  List decks = await db.select(db.decks).get();

  print(decks);
}
```

## 終わりに

Drift のWeb版の導入は公式記事だと全然情報が足りなかったので今回記事にしました。
特にココ

```diff:lib/drift/decks_database.dart
+ import 'package:drift/wasm.dart';
```

など、GitHub上で検索しないと出てこないのです。かなり困りました。
私のようにライブラリの導入だけで数日間悩んでしまわないように、この記事が誰かの役に立てば幸いです。

また、Drift は個人的に開発している[こちら](https://flash-pdf-card.web.app/)のサービスで利用しています。

https://flash-pdf-card.web.app/

まだデモ版としてWeb版のみの公開なので、モバイル版へ正式移行する際に Drift 周りのあれこれが出てくることでしょう。
その件もまた記事にできればと思います。
