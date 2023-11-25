---
title: "Flutterで使えるグラデーション付きのボタンとプログレスサークル"
emoji: "📦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "個人開発"]
published: false
---

## グラデーション付きのパーツで画面をリッチにしたい

Flutterでアプリを開発してる中で、画面をリッチにするためにボタンやプログレスサークルにグラデーションを付けたくなりました。

当時検索してもなかなか良いサンプルが見つからなかったので、コンポーネントにしたものを検索ワードと一緒に残しておきます。

## フローティングアクションボタン

フローティングアクションボタンにグラデーションを付けたものです。
アイコンと文字を両方表示できるようにしています。

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
    this.gradientColors = const [Colors.blue, Colors.purple],
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

## ボタン

画像と文字を表示したアウトラインボタンです。

### UI

![alt](https://raw.githubusercontent.com/kou72/zenn/main/image/gradient-container.png)

### コンポーネント

```dart:lib/components/gradient_container.dart
import 'package:flutter/material.dart';

class GradientContainer extends StatelessWidget {
  final String text;
  final IconData iconData;
  final double width;
  final double height;
  final List<Color> colors;
  final VoidCallback? onTap;

  const GradientContainer({
    Key? key,
    required this.text,
    required this.iconData,
    this.width = 200,
    this.height = 100,
    this.colors = const [Colors.blue, Colors.purple],
    this.onTap,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Container(
      alignment: Alignment.center,
      decoration: BoxDecoration(
        gradient: LinearGradient(
          colors: colors,
          begin: Alignment.topLeft,
          end: Alignment.bottomRight,
        ),
        borderRadius: BorderRadius.circular(24.0),
      ),
      width: width + 4,
      height: height + 4,
      child: Material(
        color: Colors.white,
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(22.0),
        ),
        child: InkWell(
          onTap: onTap,
          borderRadius: BorderRadius.circular(22.0),
          child: SizedBox(
            width: width,
            height: height,
            child: _buildIconWithText(),
          ),
        ),
      ),
    );
  }

  Widget _buildIconWithText() {
    return Column(
      mainAxisAlignment: MainAxisAlignment.center,
      children: <Widget>[
        ShaderMask(
          shaderCallback: (bounds) => LinearGradient(
            colors: colors,
            begin: Alignment.topLeft,
            end: Alignment.bottomRight,
          ).createShader(bounds),
          child: Icon(
            iconData,
            color: Colors.white,
            size: 48.0,
          ),
        ),
        const SizedBox(height: 8.0),
        ShaderMask(
          shaderCallback: (bounds) => LinearGradient(
            colors: colors,
            begin: Alignment.topLeft,
            end: Alignment.bottomRight,
          ).createShader(bounds),
          child: Text(
            text,
            style: const TextStyle(
              color: Colors.white,
              fontSize: 16.0,
            ),
          ),
        ),
      ],
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
          title: Text("Gradient Container"),
          backgroundColor: Colors.blue[100],
        ),
        body: Center(
          child: GradientContainer(
            text: "画像を選択",
            iconData: Icons.upload_file,
            width: 200,
            height: 100,
            colors: const [Colors.blue, Colors.purple],
            onTap: () => {},
          ),
        ),
      ),
    );
  }
}
```

### DartPad

https://dartpad.dev/?id=0e497721c3ea281a166ef0b75d325333

## プログレスサークル
