---
title: "[UIKit]UIScene, UIWindowScene, UIWindow"
date: 2024-08-22 12:00 +0900
categories:
  - 🍎 iOS
  - UIKit
tags:
  - UIKit
  - UIScene
---


![](/assets/img/post/2024/028.png)

iOS 13 이후 SwiftUI 발표와 함께 아이패드에서 스플릿뷰를 통해 멀티 윈도우를 지원하게 되면서,
SceneDelegate, Scene 이란 개념이 생겨났다고만 알고 있었습니다.
프로젝트를 할 때마다 window 와 scene에 대한 정확한 개념이 없어서 대충 이런거겠거니.. 하고 사용했었는데, 이번에 제대로 짚고 넘어가보려고 합니다.

## UIWindow, window

window 는 `UIWindow`의 인스턴스 입니다.
window의 역할은 아래와 같습니다.
- 앱에 표시되는(visible) contents를 포함한다.
- 뷰 및 기타 app object에 touch event 전달하는데 중요한 역할을 한다.
- app의 view controller와 상호작용하여 orientation 변경(화면 회전)을 처리한다.
- `UIKit` 에서 **event를 수신하고, 관련 event를 root view controller 및 관련 view에 전달한다.**

## Window-based (~ iOS 12)

scene을 알아보기전에 scene 이전의 구조를 알아볼 필요가 있는데요.

모든 View의 최상위 hierarchy에는 window가 있었습니다.
앱이 실행될 때 window가 생성되고, window 위에 rootViewController가 올라가는 형태였습니다.
이때 까지는 멀티 윈도우를 지원하지 않았기에, 아이폰 아이패드 모두 싱글 윈도우 앱이었고, window가 하나만 존재하는 구조였습니다.

![](/assets/img/post/2024/029.png)

> 이미지 출처: https://noah-ios.dev/window-scene/

![](/assets/img/post/2024/030.png)

![](/assets/img/post/2024/031.png)

> 이미지 출처: View Controller Programming Guide for iOS


## Window Scene-Based (iOS 13 이후)

iOS 13 이후 어떻게 바뀌었을까요?

멀티 윈도우를 지원함에 따라, 하나의 앱이 **여러개의 창**을 띄울 수 있게 되었습니다.
여기서 말하는 창은 UIScene 이 됩니다.

먼저 iOS 17.4로 빌드한 화면의 hierarchy를 확인해보겠습니다.
최상위에 UIScene이 아닌 UIWindowScene이 있네요.
이게 무슨일일까요?

![](/assets/img/post/2024/032.png)

### UIScene, UIWindowScene

UI를 보여줄 창을 만들 때 UIWindowScene 객체를 만들게 되는데요.

![](/assets/img/post/2024/033.png)

UIWindowScene 객체는 UIScene을 상속받고 있기 때문에, UIWindowScene 에서 UIScene의 메소드 및 프로퍼티에 접근이 가능합니다.

그리고 UIScene과 UIWindowScene 은 모두 delegate를 가지고 있고, scene 의 상태가 변함에 따라 해당하는 delegate 메소드를 호출함으로써 scene lifeCycle 이벤트에 적절히 반응할 수 있도록 해줍니다.

![](/assets/img/post/2024/034.png)


이것이 바로 **SceneDelegate** 입니다.
SceneDelegate 는 넘어가겠습니다.

이건 iOS 13 이전의 UI 구조이고

![](/assets/img/post/2024/035.png)

이건 iOS 13 이후 UIWindowScene 기반의 UI 구조입니다.

![](/assets/img/post/2024/036.png)

이제 조금 이해가 되는 것 같습니다 !

iOS 13 이후, 하나의 앱에 여러 개의 창을 띄울 수 있게 되었습니다.(멀티 윈도우) 
**UIScene** 이 하나의 창 역할을 하고, 구현을 할 땐 UIScene을 상속받는 **UIWindowScene** 을 통해 구현하게 됩니다.
그리고 하나의 창 (UIWindowScene)에 여러 **UIWindow** 를 띄울 수 있게 된겁니다.

### iOS12 이전에도 UIScreen 위에 UIWindow가 여러개 떠있는데 이건 멀티윈도우 아님?

멀티 윈도우의 개념을 잘 짚고 넘어가야 하는데, 여기서 말하는 멀티윈도우란
iOS13 이후 등장한 iPadOS 의 Split View 나 Slide Over를 통해 같은 앱의 여러 window를 띄울 수 있음을 설명한 것 이고,
iOS12 이전에 UIWindow를 여러개 띄우는 건 하나의 UIScreen 위에 키보드나 알림창 같은 윈도우를 오버레이할 순 있었지만, 같은 앱의 여러 윈도우를 띄울 순 없었습니다.