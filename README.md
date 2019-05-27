远程推送扩展使用说明

[TOC]

# 1.自定义通知选项  
自iOS8以后，开发者就可以自定义通知的选项，如下图所示:

## 1.1 集成自定义选项
下面的代码是iOS10之前的对通知处理的自定义选项的实例代码：

```
UIMutableUserNotificationAction *lookAction = [[UIMutableUserNotificationAction alloc] init];
lookAction.identifier = @"action.join";
lookAction.title = @"接收邀请";
lookAction.authenticationRequired = YES;

UIMutableUserNotificationAction *joinAction = [UIMutableUserNotificationAction new];
joinAction.identifier = @"action.look";
lookAction.title = @"查看邀请";
lookAction.activationMode = UIUserNotificationActivationModeForeground;

UIMutableUserNotificationAction *cancelAction = [UIMutableUserNotificationAction new];
cancelAction.identifier = @"action.cancel";
cancelAction.title = @"取消";
cancelAction.destructive = YES;

UIMutableUserNotificationCategory *category = [UIMutableUserNotificationCategory new];
category.identifier = @"selfoption";
[category setActions:@[lookAction, joinAction, cancelAction] forContext:(UIUserNotificationActionContextDefault)];

OPushNotificationConfiguration *config = [[OPushNotificationConfiguration alloc] init];
config.types = OPushAuthorizationOptionsSound|OPushAuthorizationOptionsAlert|OPushAuthorizationOptionsBadge;
config.categories = @[category];
[OdinPush setupNotification:config];
```

而iOS10对本地通知和远程通知做了统一的处理，使用UNUserNotification框架也能实现该功能。  
示例代码如下：

```
UNNotificationAction *lookAction = [UNNotificationAction actionWithIdentifier:@"action.join" title:@"接收邀请" options:UNNotificationActionOptionAuthenticationRequired];

UNNotificationAction *joinAction = [UNNotificationAction actionWithIdentifier:@"action.look" title:@"查看邀请" options:UNNotificationActionOptionForeground];
    
UNNotificationAction *cancelAction = [UNNotificationAction actionWithIdentifier:@"action.cancel" title:@"取消" options:UNNotificationActionOptionDestructive];
UNTextInputNotificationAction * inputAction = [UNTextInputNotificationAction actionWithIdentifier:@"action.input" title:@"评论" options:UNNotificationActionOptionForeground textInputButtonTitle:@"发送" textInputPlaceholder:@"请输入评论"];

UNNotificationCategory *notificationCategory = [UNNotificationCategory categoryWithIdentifier:@"selfoption" actions:@[lookAction, joinAction, cancelAction, inputAction] intentIdentifiers:@[] options:UNNotificationCategoryOptionCustomDismissAction];

// 将 category 添加到通知中心
OPushNotificationConfiguration *config = [[OPushNotificationConfiguration alloc] init];
config.types = OPushAuthorizationOptionsSound|OPushAuthorizationOptionsAlert|OPushAuthorizationOptionsBadge;
config.categories = @[notificationCategory];
[OdinPush setupNotification:config];
```

## 1.2 触发
在远程或者本地推送都可以使用参数**category**，该参数对应的值就是categoryWithIdentifier后面的参数，所以根据以上代码可以知道，开发者可以注册多套自定义选项，根据不同的category来唤起对应的自定义选项配置。

# 2.富媒体推送
自iOS10之后，苹果对远程推送做了很大的改变：  
1. 将本地推送和远程推送统一了起来；  
2. 将推送相关的代码归结到独立的框架UserNotifications，而不是附属在UIKit框架中。

富媒体示例图

## 2.1 集成Notification Service Extension

添加Notification Service Extension
如下图所示，选择自己的工程文件，点击 **+** ，添加Notification Service Extension：

新添加的Extension是以target形式添加到工程中的，因为是新的target，所以bundleId不能与项目的工程的bundleId一致，且需要给其配置对应的provision file。添加成功后，系统会自动生成NotificationService文件。当通知抵达时，系统就会调用下面的方法：

> \- (void)didReceiveNotificationRequest:(UNNotificationRequest *)request withContentHandler:(void (^)(UNNotificationContent * _Nonnull))contentHandler

开发者可以在该方法里面做一些事情：
-  对通知内容修改。例如为了保密，后台发送加密消息，开发者就在此解密；
-  加载一些媒体文件，比如视频，音频以及图片，展示到通知中；

如果开发者在上面的方法里面长时间都不调用contentHandler，那么系统就会调用下面的方法：

> \- (void)serviceExtensionTimeWillExpire

## 2.2 配置工程

选择该扩展对应的target，然后点击General，将调整Deployment Target调整为你需要适配的版本，由于该框架本身的版本不低于iOS10，所以最低版本是iOS10。

## 2.3 触发Notification Service Extension

### 2.3.1 本地通知
对于本地通知来说，不需要做额外的设置，不使用该扩展，也能达到富媒体通知的效果，只需要媒体资源即可，代码如下:

```
UNMutableNotificationContent *content = [[UNMutableNotificationContent alloc] init];
NSString *path = [NSBundle.mainBundle pathForResource:@"youth" ofType:@"jpeg"];
UNNotificationAttachment *attachment = [UNNotificationAttachment attachmentWithIdentifier:@"ident" URL:[NSURL fileURLWithPath:path] options:nil error:nil];
content.attachments = @[attachment];
content.title = @"This is a title.";
UNTimeIntervalNotificationTrigger *trigger = [UNTimeIntervalNotificationTrigger triggerWithTimeInterval:0.1 repeats:NO];
UNNotificationRequest *request = [UNNotificationRequest requestWithIdentifier:@"requestId" content:content trigger:trigger];
[UNUserNotificationCenter.currentNotificationCenter addNotificationRequest:request withCompletionHandler:^(NSError * _Nullable error) {
    
}];
```

**！！上面的代码没有走Notification Service Extension的流程！！**  

UNNotificationAttachment的资源**必须是本地资源**，无论是工程的资源还是下载到本地的资源，不能是网络链接，否则UNNotificationAttachment会创建失败。

### 2.3.2 远程推送触发
在创建APNs的payload的时候，有对应的字段**mutable-content**，将其值设置1，那么该通知就会调用Notification Service Extension。

```
{
   "aps":{
        "alert" : {
             "title" : "iOS远程消息，我是主标题！-title"
            },
        "mutable-content" : 1
    }
}
```

# 3.自定义UI通知

## 3.1 工程配置
该功能就是使用iOS10之后的Notification Content Extension，添加新的target与之前的富媒体推送设置过程类似，在此不再重复说明，添加后，系统会自动生成四个文件，可以在NotificationViewController中添加UI和逻辑代码，也可以操作MainInterface.storyboard实现UI效果。info.plist文件里面的配置很重要。  

配置具体内容如下

```
<dict>
	<key>UNNotificationExtensionDefaultContentHidden</key>
	<true/>
	<key>UNNotificationExtensionCategory</key>
	<array>
		<string>content1</string>
	</array>
	<key>UNNotificationExtensionInitialContentSizeRatio</key>
	<real>0.4</real>
</dict>
</plist>
```

NotificationViewController遵守了UNNotificationContentExtension协议，系统自动生成了协议方法：

> \- (void)didReceiveNotification:(UNNotification *)notification

在触发该扩展后，系统会调用该方法，开发者可在填充通知的内容。系统默认会将自定义的通知和系统通知都展示出来，这样一来，通知内容就可能重复了，将**UNNotificationExtensionDefaultContentHidden**设为YES，那么就能隐藏系统通知，只展示自定义通知。  

## 3.2 触发Notification Content Extension

想要系统调用该扩展，需要在该target的info.plist里面设置UNNotificationExtensionCategory，该key对应的value可以是string或者array，如果是array，array下面的应该都是string。如上面的设置所示，设置为content1，也可以设置更多值。  

在构建远程推送payload，设置category为content1，那么就可以调用该扩展。

```
{
  "aps" : {
    "alert" : {
      "title" : "iOS远程消息，我是主标题！-title"
    },
    "category" : "content1",
    "badge" : "2"
  }
}
```
而本地通知想调用该Extension，那么就需要设置**UNMutableNotificationContent**的**categoryIdentifier**的值即可。


