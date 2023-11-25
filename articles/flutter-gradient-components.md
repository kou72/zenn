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

プログレスサークルとは進捗率を円形に表示するUIです。
作ってるうちにクルクル回るバージョンと、進捗率を表示するバージョンの2種類になりました。

### UI

![alt](https://raw.githubusercontent.com/kou72/zenn/main/image/gradient-circular-spinning-indicator.gif)

### コンポーネント1

```dart:lib/components/gradient_circular_spinning_indicator.dart
import 'package:flutter/material.dart';
import 'dart:math';

class GradientCircularSpinningIndicator extends StatefulWidget {
  final double width;
  final double height;
  final List<Color> colors;
  final int milliseconds;

  const GradientCircularSpinningIndicator({
    Key? key,
    this.width = 100.0,
    this.height = 100.0,
    this.colors = const [Colors.blue, Colors.purple],
    this.milliseconds = 1500,
  }) : super(key: key);

  @override
  GradientCircularSpinningIndicatorState createState() =>
      GradientCircularSpinningIndicatorState();
}

class GradientCircularSpinningIndicatorState
    extends State<GradientCircularSpinningIndicator>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _animationStart;
  late Animation<double> _animationEnd;

  @override
  void initState() {
    super.initState();

    _controller = AnimationController(
      duration: Duration(milliseconds: widget.milliseconds),
      vsync: this,
    );

    _animationStart = Tween(begin: 0.0, end: 1.0).animate(
      CurvedAnimation(
        parent: _controller,
        curve: const Interval(0.0, 0.7, curve: Curves.fastOutSlowIn),
      ),
    );

    _animationEnd = Tween(begin: 0.0, end: 1.0).animate(
      CurvedAnimation(
        parent: _controller,
        curve: const Interval(0.2, 1.0, curve: Curves.fastOutSlowIn),
      ),
    );

    _controller.repeat();
  }

  @override
  Widget build(BuildContext context) {
    return SizedBox(
      width: widget.width,
      height: widget.height,
      child: AnimatedBuilder(
        animation: _controller,
        builder: (context, child) {
          return CustomPaint(
            painter: GradientCircularProgressPainter(
                _animationStart.value, _animationEnd.value, widget.colors),
          );
        },
      ),
    );
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
}

class GradientCircularProgressPainter extends CustomPainter {
  final double startProgress;
  final double endProgress;
  final List<Color> colors;

  GradientCircularProgressPainter(
      this.startProgress, this.endProgress, this.colors);

  @override
  void paint(Canvas canvas, Size size) {
    final Gradient gradient = LinearGradient(
      colors: colors,
    );

    final Rect rect = Rect.fromCircle(
      center: Offset(size.width / 2, size.height / 2),
      radius: size.width / 2,
    );

    final Paint paint = Paint()
      ..shader = gradient.createShader(rect)
      ..strokeWidth = 8.0
      ..style = PaintingStyle.stroke;

    final double startAngle = 2 * pi * startProgress - pi / 2;
    final double endAngle = 2 * pi * endProgress - pi / 2;
    final double sweepAngle = endAngle - startAngle;

    canvas.drawArc(
      Rect.fromCircle(
          center: Offset(size.width / 2, size.height / 2),
          radius: size.width / 2 - paint.strokeWidth / 2),
      startAngle,
      sweepAngle,
      false,
      paint,
    );
  }

  @override
  bool shouldRepaint(covariant CustomPainter oldDelegate) {
    return true;
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
          title: Text("Gradient Circular Spinning Indicator"),
          backgroundColor: Colors.blue[100],
        ),
        body: Center(
          child: GradientCircularSpinningIndicator(
            width: 100,
            height: 100,
            colors: [Colors.blue, Colors.purple],
            milliseconds: 1500,
          ),
        ),
      ),
    );
  }
}
```

### DartPad

https://dartpad.dev/?id=16d9278bfb765738fde155c0a527619b

### コンポーネント2

```dart:lib/components/gradient_circular_progress_indicator.dart
import 'package:flutter/material.dart';
import 'dart:math';

class GradientCircularProgressIndicator extends StatefulWidget {
  final double width;
  final double height;
  final List<Color> colors;
  final double progress;

  const GradientCircularProgressIndicator({
    Key? key,
    this.width = 100.0,
    this.height = 100.0,
    this.colors = const [Colors.blue, Colors.purple],
    required this.progress,
  }) : super(key: key);

  @override
  GradientCircularProgressIndicatorState createState() =>
      GradientCircularProgressIndicatorState();
}

class GradientCircularProgressIndicatorState
    extends State<GradientCircularProgressIndicator>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _animation;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      duration: const Duration(milliseconds: 500),
      vsync: this,
    );

    _animation = Tween(begin: 0.0, end: widget.progress).animate(
      CurvedAnimation(parent: _controller, curve: Curves.easeInOut),
    );

    _controller.forward();
  }

  @override
  void didUpdateWidget(covariant GradientCircularProgressIndicator oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (oldWidget.progress != widget.progress) {
      _animation = Tween(begin: oldWidget.progress, end: widget.progress)
          .animate(
              CurvedAnimation(parent: _controller, curve: Curves.easeInOut));
      _controller
        ..value = 0
        ..forward();
    }
  }

  @override
  Widget build(BuildContext context) {
    return SizedBox(
      width: widget.width,
      height: widget.height,
      child: AnimatedBuilder(
        animation: _animation,
        builder: (context, child) {
          return CustomPaint(
            painter: GradientCircularProgressPainter(
                _animation.value, widget.colors),
          );
        },
      ),
    );
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
}

class GradientCircularProgressPainter extends CustomPainter {
  final double progress;
  final List<Color> colors;

  GradientCircularProgressPainter(this.progress, this.colors);

  @override
  void paint(Canvas canvas, Size size) {
    final Gradient gradient = LinearGradient(
      colors: colors,
    );

    final Rect rect = Rect.fromCircle(
      center: Offset(size.width / 2, size.height / 2),
      radius: size.width / 2,
    );

    final Paint paint = Paint()
      ..shader = gradient.createShader(rect)
      ..strokeWidth = 8.0
      ..style = PaintingStyle.stroke;

    const double startAngle = -pi / 2;
    final double sweepAngle = 2 * pi * progress;

    canvas.drawArc(
      Rect.fromCircle(
          center: Offset(size.width / 2, size.height / 2),
          radius: size.width / 2 - paint.strokeWidth / 2),
      startAngle,
      sweepAngle,
      false,
      paint,
    );
  }

  @override
  bool shouldRepaint(covariant CustomPainter oldDelegate) {
    return true;
  }
}
```

### 使い方

```dart:lib/main.dart
class MyAppState extends State<MyApp> {
  double _progress = 0.0;

  @override
  void initState() {
    super.initState();
    Timer.periodic(Duration(seconds: 1), (timer) {
      setState(() {
        _progress += 0.1; // 0.1加算します。
      });
    });
  }

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      home: Scaffold(
        appBar: AppBar(
          title: const Text("Gradient Circular Progress Indicator"),
          backgroundColor: Colors.blue[100],
        ),
        body: Center(
          child: GradientCircularProgressIndicator(
            width: 100,
            height: 100,
            colors: const [Colors.blue, Colors.purple],
            progress: _progress,
          ),
        ),
      ),
    );
  }
}
```

### DartPad

https://dartpad.dev/?id=af177df8eb760249cae61a2ceb7421c8

## 終わりに

暗記カードアプリを個人開発しているので、よろしければ使ってみてください。

https://flash-pdf-card.web.app/

まだデモ版なので感想お待ちしております。
