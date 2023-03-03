# flutter_base_frame

flutter base frame

# 项目开始

此项目基于Getx+Flutter3 以及FlutterBoost做原生混编

## 先说下项目配置

```dart
class AppConfigManger {
  /// app环境
  static final String environment = appConfigMap['environment'];
  /// 是否显示灰度效果
  static bool isShowGrayColorFilter = appConfigMap['isShowGrayColorFilter'];
  /// app环境
  static final String appRunType = appConfigMap['appRunType'];

  /// app全局配置表
  static const Map<String,dynamic> appConfigMap = {
    'environment':'DEV',//'DEV'：开发环境 'TEST'：测试环境 'UAT'：UAT环境 'PRODUCT'：生产环境
    'isShowGrayColorFilter':false,// true：启用灰度滤镜，false:不启用灰度滤镜
    'appRunType':'FLUTTER',// 'FLUTTER'： 使用flutter启动项目，'NATIVE_MIXED'：使用原生启动项目
  };

}
```

`AppConfigManger` 进行App的全局配置，比如环境变量，灰度效果，APP类型（纯Flutter/混编）


### APP入口函数设置

```dart

void main() async {
  //初始化三方库
  ApplicationSdkManger().init();
  ApplicationSdkManger().setup();
  SystemChrome.setPreferredOrientations(
    //强制竖屏
      [DeviceOrientation.portraitUp, DeviceOrientation.portraitDown])
      .then((value) => runZonedGuarded(() => runApp(const MyApp()), (obj, error) {
    //todo 错误日志上报
  }));
}
...

/// 初始化第三方sdk
class ApplicationSdkManger {
  init() async {

    //初始化插件
    _initWidgetBanding();
    //设置抓包
    _setProxy();
    //初始化本地存储
    _initCatch();
    //初始化打印工具类
    _initLog();
    //初始化屏幕适配组件
    _initScreenUtil();

  }
  ...

}

```

由于项目需要同时兼容纯Flutter和混编，所以我们在项目的初始化会有差异

```dart

/// 只有在混编模式下需要在RunApp()之前初始化
_initScreenUtil() async{
  if(!NatureAppSettings.isFlutter) {
    await ScreenUtil.ensureScreenSize();
  }
}
/// 不同的模式使用不同的插件初始化方法
_initWidgetBanding() {
  Global.init();
}

class Global {

  static Future<void> init() async {

    if(NatureAppSettings.isFlutter) {
      /// 这个表示先进行原生端（ios android）插件注册，然后再处理后续操作，这样能保证代码运行正确。
      WidgetsBinding widgetsBinding = WidgetsFlutterBinding.ensureInitialized();
      FlutterNativeSplash.preserve(widgetsBinding: widgetsBinding);
    } else {
      CustomFlutterBinding();
    }
    await Future.wait([
      Get.putAsync<ConfigService>(() async => await ConfigService().init()),
    ]).whenComplete(() {
    });
  }
}


///创建一个自定义的Binding，继承和with的关系如下，里面什么都不用写
class CustomFlutterBinding extends WidgetsFlutterBinding
    with BoostFlutterBinding {}


```

##  [FlutterBoost](https://github.com/alibaba/flutter_boost/blob/master/README_CN.md) 模式下的根视图的创建


```dart
/*
  * 根视图的build方法
  * */
@override
Widget build(BuildContext context) {
  return NatureAppSettings.isFlutter ? _flutterRootWidget()
      : FlutterBoostApp(
    routeFactory,
    appBuilder: appBuilder,
  );
}

/*
  * 原生跳Flutter的路由生产方法
  * */
Route<dynamic>? routeFactory(RouteSettings settings, String? uniqueId) {
  FlutterBoostRouteFactory? func = BoostConfig.routerMap[settings.name];
  if (func == null) {
    return null;
  }
  return func(settings, uniqueId);
}

/*
  * 创建APP根页面
  *  home:原生打开flutter的页面 只有混编模式下使用
  * */

Widget appBuilder(Widget home) {
  return _boostRootWidget(home);
}

/*
  * 混编模式下的根组件
  * */
_boostRootWidget(Widget home) {
  var root = WillPopScope(
    onWillPop: onWillPopAction,
    child: RefreshConfiguration(
      hideFooterWhenNotFull: true, //列表数据不满一页,不触发加载更多
      child: GetMaterialApp(
        navigatorKey: navigatorKey,
        title: '',
        home: home,
        // theme: getLightTheme(),
        // darkTheme: getDarkTheme(),
        // 样式
        theme: ConfigService.to.isDarkModel ? AppTheme.dark : AppTheme.light,
        builder: (ctx, child) {
          ScreenUtil.init(ctx);
          return  MediaQuery(
              data: MediaQuery.of(ctx).copyWith(textScaleFactor: 1.0),
              child: FlutterEasyLoading(
                child: GestureDetector(
                  onTap: () {
                    FocusScopeNode focus = FocusScope.of(context);
                    if (!focus.hasPrimaryFocus &&
                        focus.focusedChild != null) {
                      FocusManager.instance.primaryFocus?.unfocus();
                    }
                  },
                  child:home,
                ),
              ));
        },
        debugShowCheckedModeBanner: true,
        translations: Translation(),//词典
        localizationsDelegates: Translation.localizationsDelegates,//代理
        supportedLocales: Translation.supportedLocales,//支持的语言种类
        locale: ConfigService.to.locale,//当前语言种类
        fallbackLocale: const Locale('zh', 'CN'),//默认语言
        // 添加一个回调语言选项，以备上面指定的语言翻译不存在
        localeResolutionCallback:
            (Locale? locale, Iterable<Locale> supportedLocales) {
          HiLog.log(locale.toString());
          return locale;
        }, //语言发生切换时的回调
      ),
    ),
  );
  return NatureAppSettings.isShowGrayColorFilter ?
  ColorFiltered(colorFilter: const ColorFilter.mode(Colors.grey, BlendMode.saturation),child: root) :
  root;
}

```

## FlutterBoost 用的是`FlutterBoostApp`初始化APP

~~~dart
FlutterBoostApp(
routeFactory,
appBuilder: appBuilder,
);
~~~

- ### `routeFactory(RouteSettings settings, String? uniqueId)`用来读取路由配置表中对应的页面

~~~dart
Route<dynamic>? routeFactory(RouteSettings settings, String? uniqueId) {
  FlutterBoostRouteFactory? func = BoostConfig.routerMap[settings.name];
  if (func == null) {
    return null;
  }
  return func(settings, uniqueId);
}
~~~


- ### 因为Boost会接管flutter使用的路由，所以需要给boost重新配置路由.
- ### 每次进行路由跳转时 都会在`routerMap` 中读取页面，并将参数记录到`Get`的parameters中，这么做的好处是 不论是 `FLUTTER` 还是  `NATIVE_MIXED`我们在页面中只会使用相同的方法读取入参

~~~dart

/*
* FlutterBoost路由配置
* */
class BoostConfig {

  /// Flutter页面路由配置Map
  static Map<String, FlutterBoostRouteFactory> routerMap = {
    RouteConfig.mainApp: (settings, uniqueId) => routeCupertinoFactory(settings, uniqueId, MainPage()),
    RouteConfig.simple: (settings, uniqueId) => routeCupertinoFactory(settings, uniqueId, TestPage()),
    RouteConfig.simple2: (settings, uniqueId) => routeCupertinoFactory(settings, uniqueId, Test2Page()),
    RouteConfig.simple3: (settings, uniqueId) => routeCupertinoFactory(settings, uniqueId, Test3Page()),
  };
  /// 初始化iOS风格的路由工厂
  static Route<dynamic> routeCupertinoFactory(settings, uniqueId,page) {
    return CupertinoPageRoute(
        settings: settings,
        builder: (_) {
          if(settings.arguments != null) {
            if (settings.arguments is Map<String, dynamic>) {
              Map<String, dynamic> map = settings.arguments as Map<String, dynamic>;
              if (map is Map<String, String>) {
                Get.parameters = map;
              }
            } else {
              Map<String, String> map = settings.arguments as Map<String, String>;
              Get.parameters = map;
            }

          }
          return page;
        });
  }
}
~~~



## Flutter的根视图创建

```dart
  /*
  * flutter模式下的根组件
  * */
  _flutterRootWidget() {
    var root = ScreenUtilInit(
      designSize: const Size(375, 667),//设计图的尺寸
      builder: (BuildContext context, Widget? child) {
        return WillPopScope(
          onWillPop:onWillPopAction,
          child: RefreshConfiguration(
            hideFooterWhenNotFull: true, //列表数据不满一页,不触发加载更多
            child: GetMaterialApp(
              navigatorKey: navigatorKey,
              title: '',
              theme: ConfigService.to.isDarkModel ? AppTheme.dark : AppTheme.light,
              builder: (ctx, child) {
                return MediaQuery(
                    data: MediaQuery.of(ctx).copyWith(textScaleFactor: 1.0),
                    child: FlutterEasyLoading(
                      child: GestureDetector(
                        onTap: () {
                          FocusScopeNode focus = FocusScope.of(context);
                          if (!focus.hasPrimaryFocus &&
                              focus.focusedChild != null) {
                            FocusManager.instance.primaryFocus?.unfocus();
                          }
                        },
                        child: child,
                      ),
                    ));
              },
              debugShowCheckedModeBanner: true,
              translations: Translation(),//词典
              localizationsDelegates: Translation.localizationsDelegates,//代理
              supportedLocales: Translation.supportedLocales,//支持的语言种类
              locale: ConfigService.to.locale,//当前语言种类
              fallbackLocale: const Locale('zh', 'CN'),//默认语言
              // 添加一个回调语言选项，以备上面指定的语言翻译不存在
              localeResolutionCallback:
                  (Locale? locale, Iterable<Locale> supportedLocales) {
                HiLog.log(locale.toString());
                return locale;
              }, //语言发生切换时的回调
              initialRoute: getRouteString(), //初始化路由
              onGenerateRoute: HomeRouter.generateRoute,
              getPages: RouteConfig.getPages, //路由列表
            ),
          ),
        );
      },
    );
    return NatureAppSettings.isShowGrayColorFilter ?
    ColorFiltered(colorFilter: const ColorFilter.mode(Colors.grey, BlendMode.saturation),child: root) :
    root;
  }
```
- Flutter初始化时需要实现初始化路由和路由列表

```dart

    ...
    initialRoute: getRouteString(), //初始化路由
    onGenerateRoute: HomeRouter.generateRoute,
    getPages: RouteConfig.getPages, //路由列表
    ...

```

- 路由跳转和传参

```dart

 /// 路由跳转
  /// routeName 路由名称 Boost 和 Flutter 使用相同的路由名称，
  /// 如果是跳转原生页面，注意原生路由名称不要和Flutter的路由名称重名
  /// parameters 路由参数

  static jumpRouteName(
      String routeName, Map<String, String> parameters, BuildContext context) {
    NatureAppSettings.isFlutter
        ? Get.toNamed(routeName,
            preventDuplicates: false, parameters: parameters)
        : Navigator.of(context)
            .pushNamed(routeName, arguments: parameters);
  }

```



## 屏幕适配 [flutter_screenutil](https://pub.flutter-io.cn/packages/flutter_screenutil) 使用参考github

# Screenutil初始化
```dart
//方式一 Flutter APP的初始化方式 
    可直接设置在根视图中
    _flutterRootWidget() {
    var root = ScreenUtilInit(
      designSize: const Size(375, 667),//设计图的尺寸
      ...
    }
//方式二 混编 的初始化方式

void main() async {
  // Add this line
  await ScreenUtil.ensureScreenSize();
  runApp(MyApp());
}
...
MaterialApp(
  ...
  builder: (ctx, child) {
    ScreenUtil.init(ctx);
    return Theme(
      data: ThemeData(
        primarySwatch: Colors.blue,
        textTheme: TextTheme(bodyText2: TextStyle(fontSize: 30.sp)),
      ),
      child: HomePage(title: 'FlutterScreenUtil Demo'),
    );
   },
)

```


## 例子

## 
~~~dart
   //如果你想显示一个矩形:
Container(
  width: 375.w,
  height: 375.h,
),

//如果你想基于宽显示一个正方形:
Container(
  width: 300.w,
  height: 300.w,
),

//如果你想基于高显示一个正方形:
Container(
  width: 300.h,
  height: 300.h,
),

//如果你想基于高或宽中的较小值显示一个正方形:
Container(
  width: 300.r,
  height: 300.r,
),

//适配字体

//输入字体大小（单位与初始化时的单位相同）
ScreenUtil().setSp(28) 
28.sp

//例子:
Column(
  crossAxisAlignment: CrossAxisAlignment.start,
  children: <Widget>[
    Text(
      '16sp, 因为设置了`textScaleFactor`，不会随系统变化.',
      style: TextStyle(
        color: Colors.black,
        fontSize: 16.sp,
      ),
      textScaleFactor: 1.0,
    ),
    Text(
      '16sp,如果未设置，我的字体大小将随系统而变化.',
      style: TextStyle(
        color: Colors.black,
        fontSize: 16.sp,
      ),
    ),
  ],
)
~~~

## 公共文件

``` dart
├── common
│   ├── extension 拓展文件
│   │   ├── ex_color.dart 扩展颜色
│   │   ├── ex_datetime.dart 扩展日期时间
│   │   ├── ex_icon.dart 扩展 Icon
│   │   ├── ex_list.dart 扩展 List
│   │   ├── ex_string.dart 扩展 String
│   │   ├── ex_widget.dart 拓展 widget 
│   │   └── index.dart extension 导包
│   ├── services 
│   │   └── config.dart 
│   ├── style
│   │   ├── color_schemes.g.dart APP主题设置
│   │   ├── colors.dart 应用颜色定义
│   │   ├── index.dart style 导包
│   │   ├── radius.dart 圆角定义
│   │   ├── space.dart 全局间距样式
│   │   ├── text.dart 全局字体样式
│   │   └── theme.dart 全局主题样式
│   ├── values  本地缓存code、image路径、svg图片路径
│   │   ├── constants.dart 
│   │   ├── images.dart
│   │   ├── index.dart
│   │   └── svgs.dart
│   └── widget 封装的组件
│       ├── banner.dart banner组件
│       ├── button.dart 按钮组件
│       ├── click_widget.dart 防止连续点击的Click 组件
│       ├── hi_card.dart 卡片组件
│       ├── icon.dart  icon组件
│       ├── image.dart image组件
│       ├── index.dart 导包
│       ├── input.dart 输入框组件
│       ├── loading_container.dart 封装了Lottie的Loading组件
│       ├── page_widget 封装了SmartRefresh & dio 网络请求的listViewPage
│       │   ├── build_listview.dart
│       │   ├── paging_controller.dart
│       │   ├── paging_params.dart
│       │   └── paging_state.dart
│       ├── text.dart 文本组件
│       └── text_banner.dart 文字banner （纵轴）
```

## 基础组件使用

## input组件

~~~dart

Widget _buildInputs() {
    return <Widget>[
      /// 文本
      const InputWidget.text(
        hintText: "文本",
      ).width(300).paddingBottom(AppSpace.listRow),

      /// 文本/边框
      const InputWidget.textBorder(
        hintText: "文本/边框",
      ).width(300).paddingBottom(AppSpace.listRow),

      /// 文本/填充/边框
      InputWidget.textFilled(
        hintText: "文本/填充/边框",
      ).width(300).paddingBottom(AppSpace.listRow),

      /// 图标/文本/填充/边框
      InputWidget.iconTextFilled(
        IconWidget.image(
          AssetsImages.cHomePng,
        ),
        hintText: "图标/文本/填充/边框",
      ).width(300).paddingBottom(AppSpace.listRow),

      /// 后缀图标/文本/填充/边框
      InputWidget.suffixTextFilled(
        IconWidget.image(
          AssetsImages.cHomePng,
        ),
        hintText: "后缀图标/文本/填充/边框",
      ).width(300).paddingBottom(AppSpace.listRow),

      /// 搜索
      InputWidget.search(
        hintText: "搜索",
        suffixIcon: InkWell(
          onTap: (){
            print('点击照相机');
          },
          child:IconWidget.image(AssetsImages.cHomePng),
        ),
      ).width(300).paddingBottom(AppSpace.listRow),

      // end
    ].toColumn();
  }
~~~

![9041677805861_ pic](https://user-images.githubusercontent.com/21218327/222607022-1538e1c5-df61-446f-b0c9-a12f67734955.jpg)


## button组件

~~~dart
Widget _buildButtons() {
    return <Widget>[
      // primary 主按钮
      ButtonWidget.primary(
        "主按钮",
        onTap: () {},
      ).paddingBottom(AppSpace.listRow),

      // secondary 次按钮
      ButtonWidget.secondary(
        "次按钮",
        onTap: () {},
      ).paddingBottom(AppSpace.listRow),

      // 文字按钮
      ButtonWidget.text(
        "文字按钮",
        textSize: 15,
        onTap: () {},
        // onTap: () async {
        //   await ConfigService.to.switchThemeModel();
        //   controller.update(["buttons"]);
        // },
      ).paddingBottom(AppSpace.listRow),

      // 图标按钮
      ButtonWidget.icon(
        IconWidget.image(
          AssetsImages.cHomePng,
          size: 30.w,
        ),
        onTap: () {},
      ).tightSize(30.w).paddingBottom(AppSpace.listRow),

      // 文字/填充 按钮
      ButtonWidget.textFilled(
        "15",
        bgColor: Get.theme.colorScheme.surfaceVariant.withOpacity(0.5),
        textSize: 12.sp,
        onTap: () {},
        height:30.w ,
        width:45.w ,
      ).paddingBottom(AppSpace.listRow),

      // 文字/填充/圆形 按钮
      ButtonWidget.textRoundFilled(
        "5",
        bgColor: Get.theme.colorScheme.surfaceVariant.withOpacity(0.4),
        borderRadius: 12.r,
        textSize: 9.sp,
        onTap: () {},
      ).tight(width: 24.w, height: 24.w).paddingBottom(AppSpace.listRow),

      // iconTextUpDown, // 图标/文字/上下
      ButtonWidget.iconTextUpDown(
        IconWidget.image(
          AssetsImages.cHomePng,
          size: 30.w,
        ),
        "Home",
        onTap: () {},
      ).paddingBottom(AppSpace.listRow),

      // iconTextOutlined, // 图标/文字/边框
      ButtonWidget.iconTextOutlined(
        IconWidget.image(
          AssetsImages.cHomePng,
          size: 30.w,
        ),
        "Home",
        onTap: () {},
      )
          .tight(
        width: 100.w,
        height: 50.w,
      )
          .paddingBottom(AppSpace.listRow),

      // iconTextUpDownOutlined, // 图标/文字/上下/边框
      ButtonWidget.iconTextUpDownOutlined(
        IconWidget.image(
          AssetsImages.cHomePng,
          size: 30.w,
        ),
        "Bike",
        onTap: () {},
      )
          .tight(
        width: 100.w,
        height: 50.h,
      )
          .paddingBottom(AppSpace.listRow),

      // textIcon, // 文字/图标
      ButtonWidget.textIcon(
        "Bike",
        IconWidget.image(
          AssetsImages.cHomePng,
          size: 30.w,
        ),
        onTap: () {},
      ).paddingBottom(AppSpace.listRow),

      //
    ].toColumn();
  }
~~~

![9061677805865_ pic](https://user-images.githubusercontent.com/21218327/222610953-621b6921-a1d0-41e4-8995-e54d4c4df40e.jpg)

## icon组件

~~~dart
Widget _buildView() {
    return ListView(
      children: [
        ListTile(
          leading: IconWidget.icon(Icons.home),
          title: const TextWidget.body1("IconWidget.icon"),
        ),
        ListTile(
          leading: IconWidget.image(AssetsImages.cBagPng),
          title: const TextWidget.body1("IconWidget.image"),
        ),
        ListTile(
          leading: IconWidget.svg(AssetsSvgs.cBagSvg),
          title: const TextWidget.body1("IconWidget.svg"),
        ),
        ListTile(
          leading: IconWidget.url(
              "https://ducafecat.oss-cn-beijing.aliyuncs.com/flutter_woo_commerce_getx_ducafecat/categories/c-man.png"),
          title: const TextWidget.body1("IconWidget.url"),
        ),
        ListTile(
          leading: IconWidget.icon(Icons.home,isDot: true,),
          title: const TextWidget.body1("IconWidget.icon带原点"),
        ),
        ListTile(
          leading: IconWidget.svg(AssetsSvgs.cBagSvg,badgeString: '99+',size: 40,),
          title: const TextWidget.body1("IconWidget.icon带数字"),
        ),
      ],
    );
  }
~~~

![9071677805866_ pic](https://user-images.githubusercontent.com/21218327/222611238-b31c708a-0924-4b63-b46e-da40adff46de.jpg)


## image组件

~~~dart
Widget _buildView() {
    return ListView(
      children: const [
        ListTile(
          leading: ImageWidget.url(
              "https://ducafecat.oss-cn-beijing.aliyuncs.com/wp-content/uploads/2022/02/90bb74497f090c48e1df1ec1ca31fb11-450x450.jpg"),
          title: TextWidget.body1("ImageWidget.url"),
        ),
        ListTile(
          leading: ImageWidget.asset(AssetsImages.cHomePng),
          title: TextWidget.body1("ImageWidget.asset"),
        ),
      ],
    );
  }
~~~

![9051677805865_ pic](https://user-images.githubusercontent.com/21218327/222611776-ccc65700-df06-4338-9f1b-51738d18daab.jpg)

## text组件

~~~dart
_buildWidget() {
    return ListView(
      children:  [
        const ListTile(title: TextWidget.title1("TextWidget.title1--标题1")),
        const ListTile(title: TextWidget.title2("TextWidget.title2--标题2")),
        const ListTile(title: TextWidget.title3("TextWidget.title3--标题3")),
        const ListTile(title: TextWidget.body1("TextWidget.body1--正文1")),
        const ListTile(title: TextWidget.body2("TextWidget.body2--正文2")),
        const ListTile(title: TextWidget.body3("TextWidget.body3--正文3")),
        ListTile(title: TextWidget.button(text: 'TextWidget.button--按钮')),
      ],
    );
  }
~~~

![9091677809339_ pic](https://user-images.githubusercontent.com/21218327/222613626-a10e4f60-5262-4e14-8dfe-413339b503cf.jpg)

## 基础组件+拓展布局例子
~~~dart
 Widget _buildView() {
      return <Widget>[
        // logo
        const ImageWidget.asset(
          AssetsImages.cgLogo,
          width: 60,
          height: 57,
        ).paddingBottom(22),

        // 标题1
        TextWidget.title2(
          "Let’s Sign You In",
          color: AppColors.onPrimary,
        ).paddingBottom(10),

        // 标题2
        TextWidget.body2(
          "Welcome back, you’ve",
          color: AppColors.onPrimary,
        ).paddingBottom(55),

        // 表单
        <Widget>[
          // username
          const TextWidget.body1(
            "Username or E-Mail",
            color: Color(0xff838383),
          ).paddingBottom(AppSpace.listRow),

          // username input
          InputWidget.iconTextFilled(IconWidget.icon(Icons.person))
              .paddingBottom(AppSpace.listRow * 2),

          // password
          const TextWidget.body1(
            "Password",
            color: Color(0xff838383),
          ).paddingBottom(AppSpace.listRow),

          // password input
          InputWidget.iconTextFilled(IconWidget.icon(Icons.lock_outline))
              .paddingBottom(29),

          // 登录按钮
          const ButtonWidget.primary(
            "Sıgn In",
            backgroundColor: Color(0xffFD8700),
            borderRadius: 18,
          ).tight(width: double.infinity, height: 57).onTap(() {
            HiCache.setString(Constants.token, "12345678");
            Get.offAllNamed(RouteConfig.mainApp);
          }),

          // 注册
          <Widget>[
            // 文字
            const TextWidget.body1(
              "Don’t have an accoun?",
              color: Color(0xff838383),
            ).paddingRight(AppSpace.listItem),

            // 注册按钮
            ButtonWidget.text(
              "Sign Up",
              textColor: const Color(0xff0274BC),
              textWeight: FontWeight.bold,
            ),
          ].toRow(
            mainAxisAlignment: MainAxisAlignment.center,
          ),
        ]
            .toColumn(
              crossAxisAlignment: CrossAxisAlignment.start,
            )
            .paddingAll(20)
            .card(
              color: Colors.white,
              radius: 35,
            ),

        // end
      ]
          .toColumn(
            mainAxisAlignment: MainAxisAlignment.center,
          )
          .paddingHorizontal(15);
    }

~~~

![9101677812246_ pic](https://user-images.githubusercontent.com/21218327/222620776-5dbf4afc-ba95-4cd6-a209-7d6a30207e21.jpg)


# 网络请求

## 目录

~~~dart
├── request 
│   ├── apis.dart 
│   ├── config.dart
│   ├── exception.dart
│   ├── exception_handler.dart
│   ├── loading.dart
│   ├── request.dart
│   ├── request_client.dart
│   └── token_interceptor.dart
~~~

- ## /apis.dart 接口地址

~~~dart

/// 接口地址
class APIS{
  static const login = "/login";
  static const test = "/test";
}

~~~
- ## /config.dart 请求配置

~~~dart
///请求配置
class RequestConfig{
  /// 请求基地址
  static String baseUrl = NatureAppSettings.baseUrl;
  /// 网络超时时间
  static const connectTimeout = 15000;
  /// http请求成功状态码
  static const successCode = 200;
}
~~~
- ## /exception.dart 异常处理

~~~dart
/// 异常处理
class ApiException implements Exception {
  static const unknownException = "未知错误";
  final String? message;
  final int? code;
  String? stackInfo;

  ApiException([this.code, this.message]);
  ...
  }
~~~
- ## /exception_handler.dart 全局异常处理

~~~dart
/// 全局异常处理
bool handleException(ApiException exception, {bool Function(ApiException)? onError}){
  //如果外部有处理异常的方法返回 true
  if(onError?.call(exception) == true){
    return true;
  }

  if(exception.code == 401 ){
    ///todo to login
    return true;
  }
  EasyLoading.showError(exception.message ?? ApiException.unknownException);

  return false;
}
~~~
- ## /loading.dart 自动管理Loading

~~~dart
/// 全局网络请求的Loading方法 自动管理Loading的显示和隐藏
Future loading( Function block, {bool isShowLoading = true}) async{
  if (isShowLoading) {
    showLoading();
  }
  try {
    await block();
  } catch (e) {
    // 不做任何处理直接向上抛出异常 由顶部的request捕获异常
    rethrow;
  } finally {
    dismissLoading();
  }
  return;
}
~~~
- ## /request.dart 最顶级的request方法

~~~dart
Future request(
  Function() block, {
  bool showLoading = true,
  bool Function(ApiException)? onError,
}) async {
  try {
    //自动管理Loading的显示和隐藏
    await loading(block, isShowLoading: showLoading);
  } catch (e) {
    // 当外部未处理异常时则在 handleException 中进行统一处理，如 401 则跳转登录页，其他错误统一弹出错误提示。
    handleException(ApiException.from(e), onError: onError);
  }
  return;
}
~~~

- ## /request_client.dart  封装的dio类

~~~dart
RequestClient requestClient = RequestClient();

class RequestClient {
  late Dio _dio;

  RequestClient() {
    print("初始化dio");
    _dio = Dio(
        BaseOptions(baseUrl: RequestConfig.baseUrl, connectTimeout: RequestConfig.connectTimeout));
    _dio.interceptors.add(TokenInterceptor());
    _dio.interceptors.add(PrettyDioLogger(
        requestHeader: true, requestBody: true, responseHeader: true));


  }

  Future<T?> request<T>(
    String url, {
    String method = "Get",
    Map<String, dynamic>? queryParameters,
    data,
    Map<String, dynamic>? headers,
    bool Function(ApiException)? onError,
  }) async {
    try {
      Options options = Options()
        ..method = method
        ..headers = headers;

      data = _convertRequestData(data);

      Response response = await _dio.request(url,
          queryParameters: queryParameters, data: data, options: options);

      return _handleResponse<T>(response);
    } catch (e) {
      // 当 onError 返回为 true 时即错误信息已被调用方处理，则不抛出异常，否则抛出异常。
      var exception = ApiException.from(e);
      if(onError?.call(exception) != true){
        throw exception;
      }
    }

    return null;
  }

~~~

- ## /token_interceptor.dart  请求拦截器

~~~dart

/// 请求拦截
/// 支持添加拦截器自定义处理请求和返回数据
/// 只需实现自定义拦截类继承 Interceptor 实现 onRequest 和 onResponse 即可。
class TokenInterceptor extends Interceptor{

  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    //TODO:HZ 当用户登录成功时设置 token等信息
    options.headers["Authorization"] = "Basic ZHhtaF93ZWI6ZHhtaF93ZWJfc2VjcmV0";
    options.headers["token"] = "Bearer ";
    // options.headers["response-status"] = 401;
    super.onRequest(options, handler);
  }

  /// 处理自定义response
  @override
  void onResponse(dio.Response response, ResponseInterceptorHandler handler) {
    super.onResponse(response, handler);
  }
}
~~~

## 例子

~~~dart

///基本使用

void login(String password) => request(() async {
  LoginParams params = LoginParams();
  params.username = "Ldies";
  params.password = password;
  UserEntity? user = await requestClient.post<UserEntity>(APIS.login, data: params);
  state.user = user;
  update();
});

/// 自定义异常处理
void loginError(bool errorHandler) => request(() async {
  LoginParams params = LoginParams();
  params.username = "Ldies";
  params.password = "654321";
  UserEntity? user = await requestClient.post<UserEntity>(APIS.login, data: params);
  state.user = user;
  print("-------------${user?.username ?? "登录失败"}");
  update();
}, onError: (e){
  state.errorMessage = "request error : ${e.message}";
  print(state.errorMessage);
  update();
  return errorHandler;
});

void loginError(bool errorHandler) => request(() async {
  LoginParams params = LoginParams();
  params.username = "Ldies";
  params.password = "654321";
  UserEntity? user = await requestClient.post<UserEntity>(APIS.login, data: params,  onError: (e){
    state.errorMessage = "request error : ${e.message}";
    print(state.errorMessage);
    update();
    return errorHandler;
  });
  //后边代码不会执行
  state.user = user;
  print("-------------${user?.username ?? "登录失败"}");
  update();
});

/// loading 显示隐藏
void loginLoading(bool showLoading) => request(() async {
  LoginParams params = LoginParams();
  params.username = "Ldies";
  params.password = "123456";
  UserEntity? user = await requestClient.post<UserEntity>(APIS.login, data: params,  );
  state.user = user;
  update();
}, showLoading: showLoading);


~~~







