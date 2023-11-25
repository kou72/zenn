---
title: "Flutterで使えるグラデーション付きのボタンとプログレスサークル"
emoji: "📦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "個人開発"]
published: false
---

## グラデーション付きのパーツで画面をリッチにしたい

Flutterでアプリを開発してる中で、画面をリッチにするためにボタンや円形の進捗バーにグラデーションを付けたくなりました。

当時検索してもなかなか良いサンプルが見つからなかったので、コンポーネントにしたものを検索ワードと一緒に残しておきます。

## グラデーションフローティングアクションボタン

フローティングアクションボタンにグラデーションを付けたものです。
アイコンと文字を両方表示できるようにしています。

### 検索ワード

- グラデーション付きフローティングアクションボタン
- グラデーション付き右下のボタン
- Gradient Floating Action Button

### UI

![alt](https://raw.githubusercontent.com/kou72/zenn/main/image/gradient-floating-action-button.png)

### コンポーネント

```dart:lib/components/gradient_floating_action_button.dart
import 'package:flutter/material.dart';

class GradientFloatingActionButton extends StatelessWidget {
  final VoidCallback onPressed;
  final IconData iconData;
  final String label;
  final List<Color> gradientColors;

  const GradientFloatingActionButton({
    Key? key,
    required this.onPressed,
    required this.iconData,
    required this.label,
    this.gradientColors = const [Colors.blue, Colors.purple], // デフォルトのグラデーション色
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Material(
      shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(30)),
      elevation: 4,
      child: Ink(
        decoration: BoxDecoration(
          gradient: LinearGradient(
            colors: gradientColors,
            begin: Alignment.topLeft,
            end: Alignment.bottomRight,
          ),
          borderRadius: BorderRadius.circular(30),
        ),
        child: FloatingActionButton.extended(
          backgroundColor: Colors.transparent,
          elevation: 4,
          icon: Icon(iconData),
          label: Text(label),
          onPressed: onPressed,
        ),
      ),
    );
  }
}
```

### 使い方

```dart:lib/main.dart
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      home: Scaffold(
        appBar: AppBar(
          title: Text("Gradient Floating Action Button"),
          backgroundColor: Colors.blue[100],
        ),
        floatingActionButton: GradientFloatingActionButton(
          onPressed: () => {},
          iconData: Icons.smart_toy,
          label: '画像からカード生成',
          gradientColors: const [Colors.blue, Colors.purple],
        ),
      ),
    );
  }
}
```

### DartPad

https://dartpad.dev/?id=29034dafc9a24e2d150c1f01d7d1b1cb
