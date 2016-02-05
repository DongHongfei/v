title: Ionic 集成 jpush(极光)推送之 IOS 篇
date: 2015-03-09 06:39:14
tags:

---

极光推送官方版的 phonegap 插件在[这里](https://github.com/jpush/jpush-phonegap-plugin)。
由于官方版插件 ios 版暂时没有打开通知的方法，所以在官方基础上修改了下，修改后的插件放在了[这里](https://github.com/DongHongfei/jpush-phonegap-plugin)，下面说明以修改后的插件为准。（感谢极光官方大神viper耐心帮助，同时也参考了下[@lanceli](http://ionichina.com/user/lanceli)大神的[cnodejs-ionic](https://github.com/lanceli/cnodejs-ionic)项目）


极光账户设置部分可以参考小和尚的[这篇](http://ionichina.com/topic/54f96e7b59a9bdef119234a1)分享。

下面主要说明项目代码部分修改。

<!-- more -->

#### 新建一个 ionic项目

`$ ionic start --id com.ionichina.ionicjpush ionic_jpush tabs`

*注：修改 id 为自己应用的 Bundle identifier*

#### 添加 IOS 平台

`$ cd ionic_jpush`   
`$ ionic platform add ios`

#### 安装插件

`$ ionic plugin add https://github.com/DongHongfei/jpush-phonegap-plugin.git`

*等待时间比较长，你也可以像小和尚[文章](http://ionichina.com/topic/54f96e7b59a9bdef119234a1)里介绍的先下载下来，再安装，但这个过程是跑不了的*

*（接下来，蛋疼的事情开始了）*

#### 修改配置

* 修改: `ionic_jpush\plugins\cn.jpush.phonegap.JPushPlugin\src\ios\PushConfig.plist`
修改对应的APP_KEY和CHANNEL(渠道)为你自己申请的值，

**注意**
确保有如下代码，不然后面 Xcode 运行会警告：

```
<key>APS_FOR_PRODUCTION</key>
<string>0</string>
```

#### 在 js 中添加通知实现

* 在 `app.js` 最后添加一个 push 工厂(参考了[@lanceli](http://ionichina.com/user/lanceli)大神的[Ccnodejs-ionic](https://github.com/lanceli/cnodejs-ionic)项目)

```
  .factory('Push', function() {
    var push;
    return {
      setBadge: function(badge) {
        if (push) {
          console.log('jpush: set badge', badge);
          plugins.jPushPlugin.setBadge(badge);
        }
      },
      setAlias: function(alias) {
        if (push) {
          console.log('jpush: set alias', alias);
          plugins.jPushPlugin.setAlias(alias);
        }
      },
      check: function() {
        if (window.jpush && push) {
          plugins.jPushPlugin.receiveNotificationIniOSCallback(window.jpush);
          window.jpush = null;
        }
      },
      init: function(notificationCallback) {
        console.log('jpush: start init-----------------------');
        push = window.plugins && window.plugins.jPushPlugin;
        if (push) {
          console.log('jpush: init');
          plugins.jPushPlugin.init();
          plugins.jPushPlugin.setDebugMode(true);
          plugins.jPushPlugin.openNotificationInAndroidCallback = notificationCallback;
          plugins.jPushPlugin.receiveNotificationIniOSCallback = notificationCallback;
        }
      }
    };
  });
```

在 app.js 的 run 函数里定义通知回调函数

*记得在 run 函数里引用 Push 先*

```
  // push notification callback
  var notificationCallback = function(data) {
    console.log('received data :' + data);
    var notification = angular.fromJson(data);
    //app 是否处于正在运行状态
    var isActive = notification.notification;

    // here add your code


    //ios
    if (ionic.Platform.isIOS()) {
      window.alert(notification);

    } else {
	//非ios(android)
    }
  };
```

在  $ionicPlatform.ready 里进行初始化

```
//初始化
Push.init(notificationCallback);
//设置别名
Push.setAlias("12345678");
```


#### 编译 IOS 项目

`$ ionic build ios`

*（接下来，更蛋疼的事情开始了）*

#### 修改配置 IOS 项目（不要问我为啥）


* 修改 AppDelegate.m，
添加

```
#import "APService.h"
#import "JPushPlugin.h"  //viper
```

`didFinishLaunchingWithOptions`函数中添加

```

    // Required
#if __IPHONE_OS_VERSION_MAX_ALLOWED > __IPHONE_7_1
    if ([[UIDevice currentDevice].systemVersion floatValue] >= 8.0) {
        //可以添加自定义categories
        [APService registerForRemoteNotificationTypes:(UIUserNotificationTypeBadge |
                                                       UIUserNotificationTypeSound |
                                                       UIUserNotificationTypeAlert)
                                           categories:nil];
    } else {
        //categories 必须为nil
        [APService registerForRemoteNotificationTypes:(UIRemoteNotificationTypeBadge |
                                                       UIRemoteNotificationTypeSound |
                                                       UIRemoteNotificationTypeAlert)
                                           categories:nil];
    }
#else
    //categories 必须为nil
    [APService registerForRemoteNotificationTypes:(UIRemoteNotificationTypeBadge |
                                                   UIRemoteNotificationTypeSound |
                                                   UIRemoteNotificationTypeAlert)
                                       categories:nil];
#endif
    // Required
    [APService setupWithOption:launchOptions];
```

`didRegisterForRemoteNotificationsWithDeviceToken`中添加

```
// Required
[APService registerDeviceToken:deviceToken];
[APService setDebugMode];
```


添加函数

```
- (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo {
    // Required
    [APService handleRemoteNotification:userInfo];
    BOOL isActive;
    if (application.applicationState == UIApplicationStateActive) {
        isActive = TRUE;
    } else {
        isActive = FALSE;
    }
    NSDictionary *dict=[[NSMutableDictionary alloc] initWithDictionary:userInfo];
    
    [dict setValue: [[NSNumber alloc] initWithBool:isActive] forKey:@"isActive" ];
    
    [[NSNotificationCenter defaultCenter] postNotificationName:kJPushPlugReceiveNotificaiton
                                                        object:dict] ;//viper
    
    
}
```

#### OC 代码算是完事儿，然后就是配置
* 修改项目 Capabilities，打开 Background Modes，勾选最后一项Remote notications
* 设置证书，这个就不教了，网上一大堆
* Xcode 这边就算配置完了


接下就是设置一些Xcode常规操作，编译运行，从极光官方控制台发送一条通知，然后查看Xcode控制台，应该就会有推送的通知数据打印了。
下面的事儿你自己应该搞的定。

示例代码我放在了[这里](https://github.com/DongHongfei/ionic_jpush)。