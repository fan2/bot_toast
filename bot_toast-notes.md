
## 代码结构分析

```Dart
// lib/bot_toast.dart

library bot_toast;

export 'src/basis.dart';
export 'src/bot_toast_init.dart' show BotToastInit;
export 'src/toast.dart';
export 'src/toast_navigator_observer.dart';
```

## BotToastInit

BotToastInit 返回 TransitionBuilder，构造 BotToastManager：

```Dart
// lib/src/bot_toast_init.dart

// ignore: non_constant_identifier_names
TransitionBuilder BotToastInit() {
  //确保提前初始化,保证WidgetsBinding.instance.addObserver(this);的顺序

  //ignore: unnecessary_statements
  BotToastWidgetsBindingObserver._singleton;
  return (_, Widget? child) {
    return BotToastManager(key: _key, child: child!);
  };
}
```

### 全局初始化

作为 MyApp.build MaterialApp 的 builder：

```Dart
// example/lib/main.dart

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      builder: BotToastInit(),
      title: 'BotToast Demo',
      navigatorObservers: [BotToastNavigatorObserver()],
      home: EnterPage(),
    );
  }
}
```

这里通过 builder 初始化方式可以参考：

- [flutter_easyloading](https://pub.dev/packages/flutter_easyloading) fork [github](https://github.com/fan2/flutter_easyloading)  
- [flutter_smart_dialog](https://pub.dev/packages/flutter_smart_dialog) fork [github](https://github.com/fan2/flutter_smart_dialog)  

## BotToastManager

BotToastManager 继承自 StatefulWidget。

```Dart
class BotToastManager extends StatefulWidget {
  final Widget child;

  const BotToastManager({Key? key, required this.child}) : super(key: key);

  @override
  BotToastManagerState createState() => BotToastManagerState();
}
```

BotToastManagerState 中的 `_map` 存储了字符串（groupKey）到字典的映射，二维字典记录从 UniqueKey 到 _IndexWidget 的映射。
`_pending` 为 UniqueKey 的集合。

```Dart
class BotToastManagerState extends State<BotToastManager> {
  final Map<String, Map<UniqueKey, _IndexWidget>> _map = <String, Map<UniqueKey, _IndexWidget>>{};

  final Set<UniqueKey> _pending = <UniqueKey>{};

  int _nextAddIndex = 0;

  List<_IndexWidget> get _children =>
      _map.values.fold<List<_IndexWidget>>(<_IndexWidget>[], (value, items) {
        return value..addAll(items.values);
      })
        ..sort((a, b) => a.index.compareTo(b.index));

  @override
  Widget build(BuildContext context) {
    return Stack(
      children: <Widget>[
        widget.child,
      ]..addAll(_children),
    );
  }

  void insert(String groupKey, UniqueKey key, Widget widget) {

  }
```

### botToastManager

botToastManager 返回 BotToastManagerState。

```Dart
// lib/src/bot_toast_init.dart

final GlobalKey<BotToastManagerState> _key = GlobalKey<BotToastManagerState>();

BotToastManagerState get botToastManager {
  assert(_key.currentState != null);
  return _key.currentState!;
}
```

## BotToast

### showCustomText

BotToast.showText -> showCustomText(toastBuilder: (_) => TextToast)  

> BotToast.showCustomText -> showAnimationWidget  

### showCustomLoading

BotToast.showLoading -> showCustomLoading(toastBuilder: (_) => const LoadingWidget())，可选动画参数 WrapAnimation(wrapAnimation,wrapToastAnimation)  

> BotToast.showCustomLoading -> showAnimationWidget  

### showAnimationWidget

*showCustomText* 和 *showCustomLoading* 最终都调用 `showAnimationWidget`。

showAnimationWidget(toastBuilder: toastBuilder):

- required ToastBuilder toastBuilder;  
- 可选动画参数 WrapAnimation(wrapAnimation,wrapToastAnimation)。  

*showAnimationWidget* 调用 `showEnhancedWidget`，传递 warpWidget 和 toastBuilder 等参数。

#### showEnhancedWidget

warpWidget 和 toastBuilder，最终调用 `showWidget`。

> botToastManager.insert

```
  ///显示一个Widget在屏幕上,该Widget可以跨多个页面存在
  ///
  ///[toastBuilder] 生成需要显示的Widget的builder函数
  ///[key] 代表此Toast的一个凭证,凭此key可以删除当前key所定义的Widget,[remove]
  ///[groupKey] 代表分组的key,主要用于[removeAll]和[remove]
  ///[CancelFunc] 关闭函数,主动调用将会关闭此Toast
  ///这是个核心方法
  static CancelFunc showWidget(
      {required ToastBuilder toastBuilder, UniqueKey? key, String? groupKey}) {
    final gk = groupKey ?? defaultKey;
    final uniqueKey = key ?? UniqueKey();
    final CancelFunc cancelFunc = () {
      remove(uniqueKey, gk);
    };

    botToastManager.insert(gk, uniqueKey, toastBuilder(cancelFunc));
    return cancelFunc;
  }
```

### insert-update-build

botToastManager(`BotToastManagerState`) insert 流程：

1. widget 依次封装到 ProxyInitState、ProxyDispose、_IndexWidget；  
2. 设置字典：_map[groupKey]![key] = _IndexWidget;  
3. _pending.add(key) && _update();  

```Dart
// lib/src/bot_toast_manager.dart

  void _update() {
    if (mounted) {
      setState(() {});
    }
  }
```

_update->setState 会调用 BotToastManageState.build 重绘，其中 `_children` 为从 _map 中取 List<_IndexWidget>。
