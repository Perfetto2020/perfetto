# Navigator 与 back

flutter 是通过 Navigator 与 route 来进行页面的切换的。navigator.push 推入一个 route，navigator.pop 退出一个页面。

如何进行页面的返回呢？

最基本的当然就是 navigator.pop，把顶部 route 从栈中 pop 出去，露出下面的一个 route，实现返回的效果。

## 两个问题：

1. 系统的返回是如何响应的？所谓系统返回是指 Android 的实体返回键或者虚拟返回键，或者 Android，iOS 都有的 back swipe。
2. 如何拦截系统 back 事件

### 如何响应系统的返回

在 flutter/lib/src/services/system_channels.dart 中定义了一个叫 navigation 的 MethodChannel，这个 channel 就是用来传递原生平台的 back 事件的。

```dart
static const MethodChannel navigation = MethodChannel(
    'flutter/navigation',
    JSONMethodCodec(),
);
```

在 mixin WidgetsBinding 中定义了这个 method channel 的 call handler：

```dart
	void initInstances() {
	  ...
  	SystemChannels.navigation.setMethodCallHandler(_handleNavigationInvocation);
		...
	}

  Future<dynamic> _handleNavigationInvocation(MethodCall methodCall) {
    switch (methodCall.method) {
      case 'popRoute':
        return handlePopRoute();
      case 'pushRoute':
        return handlePushRoute(methodCall.arguments as String);
    }
    return Future<dynamic>.value();
  }
```

在 handlePopRoute 中会通知 WidgetsBinding 中注册 observer

```dart
Future<void> handlePopRoute() async {
  for (final WidgetsBindingObserver observer in List<WidgetsBindingObserver>.from(_observers)) {
    if (await observer.didPopRoute())
      return;
  }
  /// 如果 Flutter App 内没有消费掉 back 时间，那么
  /// Removes the topmost Flutter instance, presenting what was before it.
  /// 在 Android 上就是从 Activity stack 中移除当前 Activity
  /// 在 iOS 上就是 popViewControllerAnimated 或 dismissViewControllerAnimated
  SystemNavigator.pop();
}

  static Future<void> pop({bool animated}) async {
    await SystemChannels.platform.invokeMethod<void>('SystemNavigator.pop', animated);
  }
```

谁往 WidgetsBinding 中注册了 observer 呢？MaterialApp/CupertinoApp 的 child WidgetsApp

在 widgets/app.dart 的 _WidgetsAppState：

```dart
  void initState() {
    super.initState();
    _updateNavigator();
    _locale = _resolveLocales(WidgetsBinding.instance.window.locales, widget.supportedLocales);
    WidgetsBinding.instance.addObserver(this);
  }

  Widget build(BuildContext context) {
    Widget navigator;
    navigator = Navigator(...)
    ...
  }

  // On Android: the user has pressed the back button.
  @override
  Future<bool> didPopRoute() async {
    assert(mounted);
    final NavigatorState navigator = _navigator?.currentState;
    if (navigator == null)
      return false;
    return await navigator.maybePop();
  }        
```

可以看出 WigetsApp 负责了 Navigator 的创建并通知 Navigator 对系统的返回事件进行响应。

### 如何拦截系统 back 事件

有时候需要拦截 back 事件，做些特殊处理。比如填写表单过程中，比如返回需要做些UI的变化

要想拦截系统的 back 事件，需要先了解 Navigator 处理 back 的流程，从这个流程中找到一个最优的 hook 的点。

## Navigator 的结构

Navigator 是个非常复杂的 Widget（navigator.dart 有 4k+ 行代码），只有了解了 Navigator 的结构才能更好的了解 Navigator 如何处理返回事件。

<img src="./a.png"/>



## Navigator 如何处理 back

从前面 Flutter 如何响应系统返回事件的讨论可以看出，系统的返回事件最终会传递到 Navigator.maybePop；另外，我们在自己处理页面（Route）的返回时，会直接使用 Navigator.pop。分这两种情况来分析

### Navigator.mayBePop



### Navigator.pop

## InsightBank中的诉求

1. 某些页面在返回时弹窗，确定返回，取消留在当前页面
2. 退出页面时执行某个操作，例如保存一个数据到redux的state tree
3. 支持页面内的简单route
4. 返回两次退出应用





LocalHistoryEntry问题跟踪

```dart
@override
void onResume() {
  super.onResume();
  if (CustomTextInputType.isCustomTextInputType(widget.textInputType)) {
    KeyboardManager.interceptInput(context);
  }
}
```





为什么把WillPopScope放到AppBar里面

cons：感觉上比较奇怪

pros：

1. 防止page复用时，有重复的onWillPop被触发。A page可以单独使用，也会嵌入到B page中使用
2. 不用每个State都去with ExitStateMixin。一个AppBar的参数可以搞定





Todo: lib/widgets/container/kyc/address_page.dart:189 forbidBack删掉

为什么弹窗能被logout的popuntil取消掉。

如何在maybePop的时候也传递返回值 T result