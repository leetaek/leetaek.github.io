+++
title = "[SwiftUI] AppDelegate 없이 Remote Push Notification 설정하기"
date = 2023-03-16T13:32:46+09:00
draft = false
tags = ["ios", "swiftui", "uikit"]
categories = ["iOS"]
tistory = "https://leetaek.tistory.com/71"
+++

- `SwiftUI`로 넘어오면서 `AppDelegate`와 `SceneDelegate`가 중심이 되던 앱의 라이프사이클에 변화가 생겼다. → [[SwiftUI]AppDelegate, SceneDelegate 만들기](https://leetaek.tistory.com/67)

- Remote Push Notification의 경우 설정을 AppDelegate에서 해줘야 했는데, 이를 순수하게 SwiftUI의 라이프 사이클에 맞게 변경해보고 싶었다.

- 아래는 결론 도출을 위해 겪은 과정이다. FCM 을 통해 Remote Notification을 구현하려고 했는데, Firebase나 인증서와 관련된 내용은 생략하고 여기에는 앱의 코드만 작성한다.

- 결론부터 말하자면 그런거 없다.🥲 **AppDelegate를 만들어 사용해야 한다. (SwiftUI 4.0 기준)**

# 코드

### @main App

```swift
import SwiftUI
import FirebaseCore

@main
struct SwiftUI_SampleApp: App {
  var notificationService = NotificationsService()

  init() {
    UIApplication.shared.delegate = NotificationsService.shared
    FirebaseApp.configure()
    notificationService.setDelegate()
    notificationService.requestPushPermission()
  }

  var body: some Scene {
    WindowGroup {
      ContentView()
    }
  }
}
```

- Notification 설정과 앱에 진입할 때에 필요한 메소드들을 NotificationService라는 싱글톤 class로 만들었다. 해당 클래스는 `UIApplication.shared.delegate`로 넘겨줘 앱에 들어올 때에 사용자 이벤트의 라우팅을 처리할 것이다.

- 기존에 AppDelegate의 `didFinishLaunchingWithOption`에서 수행했던 작업들을 init() 에서 처리해 앱을 초기화할 때 비슷하게 실행한다.

### NotificationService

```swift
// UNUserNotificationCenterDelegate의 메소드 구현
class NotificationsService: NSObject, UNUserNotificationCenterDelegate {
  static let shared = NotificationsService()
  let gcmMessageIDKey = "gcm.message_id"

	// 1.
  func requestPushPermission() {
    UNUserNotificationCenter.current().delegate = self

    let authOptions: UNAuthorizationOptions = [.alert, .badge, .sound]
    UNUserNotificationCenter.current().requestAuthorization(options: authOptions) { granted, error in
      if granted {
        print("User granted Notification Permissions")
      } else if let error = error {
        print("에러: \(error)")
      }
    }

    DispatchQueue.main.async {
      UIApplication.shared.registerForRemoteNotifications()
    }
  }

	// 2.
	func userNotificationCenter(_ center: UNUserNotificationCenter, willPresent notification: UNNotification) async -> UNNotificationPresentationOptions {
	    let userInfo = notification.request.content.userInfo

	    if let messageID = userInfo[gcmMessageIDKey] {
	      print("Message ID: \(messageID)")
	    }

	    print(userInfo)
	    return [.banner, .list, .sound]
	  }

	// 3.
  /// 백그라운드에서 받은 Push 데이터 처리
  func userNotificationCenter(_ center: UNUserNotificationCenter, didReceive response: UNNotificationResponse) async {
    let userInfo = response.notification.request.content.userInfo
    if let messageID = userInfo[gcmMessageIDKey] {
      print("Message ID: \(messageID)")
    }
    print(userInfo)
  }
}
```

1. func requestPushPermission() : 사용자에게 Notification 권한을 얻는 작업을 수행하고, `APNs`에 `deviceToken`을 등록하는 절차를 수행한다.

1. [func userNotificationCenter( … )](https://developer.apple.com/documentation/usernotifications/unusernotificationcenterdelegate/1649518-usernotificationcenter/): 앱이 foreground 상태일 때 받은 push를 처리한다.

1. [func userNotificationCenter( … )](https://developer.apple.com/documentation/usernotifications/unusernotificationcenterdelegate/1649501-usernotificationcenter/): 앱이 background 상태일 때에 받은 push를 처리한다.

```swift
extension NotificationsService: MessagingDelegate {
  // 4.
  func setDelegate() {
    Messaging.messaging().delegate = self
  }

	// 5.
  func messaging(_ messaging: Messaging,
                 didReceiveRegistrationToken fcmToken: String?) {
    print("Firebase registration token: \(String(describing: fcmToken))")
    let dataDict: [String: String] = ["token": fcmToken ?? ""]
    NotificationCenter.default.post(name: Notification.Name("FCMToken"), object: nil, userInfo: dataDict)
  }
}
```

1. func setDelegate(): FIRMessaging의 delegate 속성 설정

1. [func messaging( … )](https://firebase.google.com/docs/cloud-messaging/ios/client?hl=ko#monitor-token-refresh) : 앱 시작 시에 Firebase 등록 토큰을 전달받는 메소드

```swift
extension NotificationsService: UIApplicationDelegate {
	// 6.
  /// 앱이 종료되었을 때에 노티를 탭한 경우 여기서 수신
  func application(_ application: UIApplication,
                   didReceiveRemoteNotification userInfo: [AnyHashable : Any]) async -> UIBackgroundFetchResult {
    if let messageID = userInfo[gcmMessageIDKey] {
      print("Message ID: \(messageID)")
    }
    print(userInfo)
    return .newData
  }

	// 7.
  func application(_ application: UIApplication,
                   didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
    print("앱이 APNs에 성공적으로 등록" )
    Messaging.messaging().apnsToken = deviceToken
  }
}
```

1. [func application( …. )](https://developer.apple.com/documentation/appkit/nsapplicationdelegate/1428430-application/) : 앱이 종료되어있을 때에 노티를 탭한 경우 여기서 수신한다.

1. **[func applicaion ( … )](https://developer.apple.com/documentation/uikit/uiapplicationdelegate/1622958-application/) :** APNs에서 deviceToken에 등록 요청에 대한 응답이 오면 firebase에 해당 토큰을 등록한다.

### 실행 결과

> **2023-03-16 10:02:56.008498+0900 SwiftUI_SampleApp[3077:43561] 9.6.0 - [FirebaseMessaging][I-FCM002022] APNS device token not set before retrieving FCM Token for Sender ID '\(sender ID)'. Notifications to this FCM Token will not be delivered over APNS.Be sure to re-retrieve the FCM token once the APNS device token is set.**

- 해당 코드를 작성한 후 앱을 처음 실행시켜보면 다음과 같은 로그가 뜬다. (안뜨면 삭제후 재설치) APNs 토큰이 등록되지 않았다는 소리. →**위에서 작성한 7번 메소드가 실행되지 않았다.**

- 1번 메소드에서` UIApplication.shared.registerForRemoteNotifications()`가 실행되고, 7번 메소드가 실행되어야 하는데 해당 메소드가 실행되지 않았다. → NotificationService를 `@UIApplicationDelegateAdaptor`로 선언하면 실행된다. 아마 AppDelegate로 설정되지 않아 생기는 문제인 것 같다.

```swift
// @main App
@UIApplicationDelegateAdaptor private var NotificationDelegate: NotificationsService
```

- UIKit의 라이프스타일을 사용하지 않으려고 해당 메소드를 사용하지 않고 Firebase로 deviceToken을 넘겨주는 방법을 찾아봤는데, appDelegate 메소드를 사용하지 않으면 deviceToken을 얻기 어려운 것 같다. 아이디어 있으면 좀 알려주세요 …🥲

# 결론

- SwiftUI가 나온지 4년이 넘어가는 시점에서 아직 SwiftUI 의 앱 라이프사이클 만으로는 앱의 핵심 기능중 하나인 Remote Push Notification 을 구현할 수 없다는게 아쉽다.

- `@UIApplicationDelegateAdaptor`의 개발자 문서 항목을 들어가면 예로 나와있는 것이 AppDelegate로 DeviceToken을 얻는 법인데, 이를 보면 SwiftUI 환경으로 개발을 한다 하더라도 여전히 UIKit의 도움을 받는 것은 필수인 것 같다.

# Reference

[Registering your app with APNs | Apple Developer Documentation](https://developer.apple.com/documentation/usernotifications/registering_your_app_with_apns)

[UIApplicationDelegateAdaptor | Apple Developer Documentation](https://developer.apple.com/documentation/swiftui/uiapplicationdelegateadaptor/)

[Apple 플랫폼에서 Firebase 클라우드 메시징 클라이언트 앱 설정](https://firebase.google.com/docs/cloud-messaging/ios/client?hl=ko)
