# 沉浸式(顶天立地)

坑爹的Flutter官方居然不让Android的底部导航栏像iPhone X的底部横杆那样纳入绘制区域

MainActivity.java

```java
package io.github.v7lin.demo;

import android.app.Activity;
import android.graphics.Color;
import android.os.Build;
import android.os.Bundle;
import android.view.View;

import androidx.annotation.NonNull;

import io.flutter.embedding.android.FlutterActivity;
import io.flutter.embedding.engine.FlutterEngine;
import io.flutter.embedding.engine.plugins.FlutterPlugin;
import io.flutter.embedding.engine.plugins.activity.ActivityAware;
import io.flutter.embedding.engine.plugins.activity.ActivityPluginBinding;
import io.flutter.plugin.common.MethodCall;
import io.flutter.plugin.common.MethodChannel;
import io.flutter.plugin.platform.PlatformPlugin;
import io.flutter.plugins.GeneratedPluginRegistrant;

public class MainActivity extends FlutterActivity {

    private int systemUiVisibility = PlatformPlugin.DEFAULT_SYSTEM_UI;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            getWindow().setStatusBarColor(Color.TRANSPARENT);
        }
    }

    @Override
    public void configureFlutterEngine(@NonNull FlutterEngine flutterEngine) {
        GeneratedPluginRegistrant.registerWith(flutterEngine);
        flutterEngine.getPlugins().add(new Utility());
    }

    @Override
    protected void onPause() {
        super.onPause();
        systemUiVisibility = getWindow().getDecorView().getWindowSystemUiVisibility();
    }

    @Override
    public void onPostResume() {
        super.onPostResume();
        /**
         * {@link FlutterActivity#onPostResume()}里会调用{@link PlatformPlugin#updateSystemUiOverlays()}重置SystemUiVisibility，故而这里需要还原现场
         */
        getWindow().getDecorView().setSystemUiVisibility(systemUiVisibility);
    }
}

class Utility implements FlutterPlugin, ActivityAware, MethodChannel.MethodCallHandler {

    private MethodChannel channel;
    private Activity activity;

    @Override
    public void onAttachedToEngine(FlutterPluginBinding binding) {
        channel =
                new MethodChannel(
                        binding.getBinaryMessenger(), "v7lin.github.io/utility");
        channel.setMethodCallHandler(this);
    }

    @Override
    public void onDetachedFromEngine(FlutterPluginBinding binding) {
        channel.setMethodCallHandler(null);
        channel = null;
    }

    @Override
    public void onAttachedToActivity(ActivityPluginBinding binding) {
        activity = binding.getActivity();
    }

    @Override
    public void onDetachedFromActivityForConfigChanges() {
        onDetachedFromActivity();
    }

    @Override
    public void onReattachedToActivityForConfigChanges(ActivityPluginBinding binding) {
        onAttachedToActivity(binding);
    }

    @Override
    public void onDetachedFromActivity() {
        activity = null;
    }

    @Override
    public void onMethodCall(MethodCall call, MethodChannel.Result result) {
        switch (call.method) {
            case "setImmersiveLayout":
                if (activity != null && !activity.isFinishing()) {
                    activity.getWindow().getDecorView().setSystemUiVisibility(PlatformPlugin.DEFAULT_SYSTEM_UI | View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION);
                }
                result.success(null);
                break;
            case "setFullScreenLayout":
                if (activity != null && !activity.isFinishing()) {
                    boolean fullScreen = (boolean) call.arguments;
                    int systemUiVisibility = activity.getWindow().getDecorView().getWindowSystemUiVisibility();
                    if (fullScreen) {
                        int enabledOverlays = systemUiVisibility;
                        enabledOverlays |= View.SYSTEM_UI_FLAG_FULLSCREEN;
                        enabledOverlays |= View.SYSTEM_UI_FLAG_HIDE_NAVIGATION;
                        enabledOverlays |= View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY;
                        activity.getWindow().getDecorView().setSystemUiVisibility(enabledOverlays);
                    } else {
                        int enabledOverlays = systemUiVisibility;
                        enabledOverlays &= ~View.SYSTEM_UI_FLAG_FULLSCREEN;
                        enabledOverlays &= ~View.SYSTEM_UI_FLAG_HIDE_NAVIGATION;
                        enabledOverlays &= ~View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY;
                        activity.getWindow().getDecorView().setSystemUiVisibility(enabledOverlays);
                    }
                }
                result.success(null);
                break;
            default:
                result.notImplemented();
                break;
        }
    }
}
```

utility.dart

```dart
import 'dart:io';

import 'package:flutter/services.dart';

class Utility {
  Utility._();

  static const MethodChannel _channel =
      MethodChannel('v7lin.github.io/utility');

  static Future<void> setImmersiveLayout() {
    assert(Platform.isAndroid);
    return _channel.invokeMethod('setImmersiveLayout');
  }

  static Future<void> setFullScreenLayout(bool fullScreen) {
    assert(fullScreen != null);
    if (Platform.isIOS) {
      if (fullScreen) {
        return SystemChrome.setEnabledSystemUIOverlays(<SystemUiOverlay>[]);
      } else {
        return SystemChrome.setEnabledSystemUIOverlays(SystemUiOverlay.values);
      }
    }
    return _channel.invokeMethod('setFullScreenLayout', fullScreen);
  }
}
```

metrics_reactor.dart

```dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:flutter/widgets.dart';
import 'package:novel/utility.dart';

/// Android Only
class MetricsReactor extends StatefulWidget {
  const MetricsReactor({
    Key key,
    @required this.builder,
    this.navigationBarColor,
  }) : super(key: key);

  final WidgetBuilder builder;
  final Color navigationBarColor;

  @override
  State<StatefulWidget> createState() {
    return _MetricsReactorState();
  }
}

class _MetricsReactorState extends State<MetricsReactor>
    with WidgetsBindingObserver {
  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);
    _metricsFixed();
  }

  Future<void> _metricsFixed() async {
    SystemChrome.setSystemUIOverlayStyle(SystemUiOverlayStyle(
      systemNavigationBarColor:
          widget.navigationBarColor ?? Colors.transparent.withOpacity(0.2),
    ));
    await Utility.setImmersiveLayout();
  }

  @override
  void didChangeMetrics() {
    super.didChangeMetrics();
    setState(() {
      // do nothing
    });
  }

  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(this);
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    MediaQueryData mediaQuery = MediaQuery.of(context);
    return MediaQuery(
      data: mediaQuery.copyWith(
        padding: mediaQuery.padding.copyWith(
          bottom: mediaQuery.viewInsets.bottom,
        ),
        viewPadding: mediaQuery.viewPadding.copyWith(
          bottom: mediaQuery.viewInsets.bottom,
        ),
//        // 正确做法
//        viewInsets: mediaQuery.viewInsets.copyWith(
//          bottom: math.max(0, mediaQuery.viewInsets.bottom - _originMediaQuery.viewInsets.bottom),
//        ),
      ),
      child: Builder(
        builder: widget.builder,
      ),
    );
  }
}
```

main.dart

```dart
import 'dart:io';

import 'package:flutter/material.dart';
import 'package:novel/metrics_reactor.dart';

void main() => runApp(MyApp());

class MyApp extends StatelessWidget {
  // This widget is the root of your application.
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      theme: ThemeData(
        // This is the theme of your application.
        //
        // Try running your application with "flutter run". You'll see the
        // application has a blue toolbar. Then, without quitting the app, try
        // changing the primarySwatch below to Colors.green and then invoke
        // "hot reload" (press "r" in the console where you ran "flutter run",
        // or simply save your changes to "hot reload" in a Flutter IDE).
        // Notice that the counter didn't reset back to zero; the application
        // is not restarted.
        primarySwatch: Colors.blue,
        platform: TargetPlatform.iOS,
      ),
      home: MyHomePage(title: 'Flutter Demo Home Page'),
      builder: (BuildContext context, Widget child) {
        if (Platform.isAndroid) {
          return MetricsReactor(
            builder: (BuildContext context) {
              return MediaQuery(
                data: MediaQuery.of(context).copyWith(
                  textScaleFactor: 1.0,
                  boldText: true,
                ),
                child: child,
              );
            },
            navigationBarColor: Colors.transparent,
          );
        }
        return MediaQuery(
          data: MediaQuery.of(context).copyWith(
            textScaleFactor: 1.0,
            boldText: true,
          ),
          child: child,
        );
      },
    );
  }
}

class MyHomePage extends StatefulWidget {
  MyHomePage({Key key, this.title}) : super(key: key);

  // This widget is the home page of your application. It is stateful, meaning
  // that it has a State object (defined below) that contains fields that affect
  // how it looks.

  // This class is the configuration for the state. It holds the values (in this
  // case the title) provided by the parent (in this case the App widget) and
  // used by the build method of the State. Fields in a Widget subclass are
  // always marked "final".

  final String title;

  @override
  _MyHomePageState createState() => _MyHomePageState();
}

class _MyHomePageState extends State<MyHomePage> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Colors.red,
      appBar: AppBar(
        title: Text(widget.title),
        backgroundColor: Colors.transparent,
      ),
      body: ListView.builder(
        itemBuilder: (BuildContext context, int index) {
          return ListTile(
            title: Text('Index: $index'),
          );
        },
        itemCount: 100,
      ),
      floatingActionButton: FloatingActionButton(
        child: Text('+'),
        onPressed: () {},
      ),
      floatingActionButtonLocation: FloatingActionButtonLocation.endFloat,
      bottomNavigationBar: BottomAppBar(
        child: Container(
          height: kBottomNavigationBarHeight,
        ),
      ),
    );
  }
}
```
