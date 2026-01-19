---
title: "[SwiftUI] Destination Enum으로 타입 안전한 네비게이션 구축하기"
date: 2026-01-19 16:00 +0900
categories:
  - 🍎 iOS
  - SwiftUI
tags:
  - SwiftUI
  - Navigation
  - Architecture
  - iOS17
  - Observable
---

SwiftUI로 복잡한 네비게이션을 구현하다 보면 런타임에 화면 전환이 실패하거나, 보일러플레이트 코드가 과도하게 늘어나는 문제를 경험하게 됩니다. 이 글에서는 프로토콜 기반 네비게이션 방식에서 **Destination Enum 기반 아키텍처**로 전환하여 이러한 문제들을 해결한 경험을 공유합니다.

## 🤔 기존 네비게이션 방식의 문제점

### 프로토콜 기반 접근의 한계

사이드 프로젝트를 진행하며 기존에는 `Navagatable`와 `NavigatableViewModel` 프로토콜을 사용한 네비게이션 구조를 구현하여 사용했습니다. 이 방식은 크게 두 가지 문제를 가지고 있었습니다.

#### 1. 필수적인 ViewModel 요구사항 → 보일러플레이트 증가

```swift
// ❌ 기존 방식: 모든 화면에 ViewModel이 필수
public protocol Navigatable: AnyObject {
  var path: NavigationPath { get set }
}

open class NavigatableViewModel: HashableObject {
  public weak var navigator: (any Navigatable)?
}

class EmailSignInViewModel: NavigatableViewModel {
	// ... 네비게이션을 위한 보일러플레이트 코드
    func navigateToEmailSignUp() {
	    let viewModel = EmailSignUpViewModel(email: email)
	    viewModel.setNavigator(navigator)
	    navigator?.navigate(viewModel)
  }
}
```

단순한 화면도 반드시 ViewModel을 만들어야 했고, 네비게이션을 위한 코드가 반복되었습니다.

#### 2. 런타임 네비게이션 실패

가장 치명적인 문제는 **런타임에 목적지 화면을 찾지 못해 화면 전환에 실패**하는 경우였습니다.
구현이 누락되면  컴파일 타임에 잡히지 않고, 런타임에 화면 전환이 실패하여 빈 화면으로 이동하는 경우가 발생했습니다.

### 새로운 접근이 필요했던 이유

이러한 문제들을 해결하기 위해 다음 목표를 설정했습니다:

1. **보일러플레이트 제거**: ViewModel 없이도 네비게이션 가능해야 함
2. **컴파일 타임 안전성**: 런타임 실패를 근본적으로 차단
3. **약한 결합**: 화면 간 직접 의존성 제거
4. **중앙화된 상태 관리**: 네비게이션 로직을 한 곳에서 제어
5. **테스트 가능성**: 네비게이션 로직을 독립적으로 테스트

## 💡 해결 과정: Destination 기반 값 네비게이션

### Destination Enum: 네비게이션을 값으로 표현

핵심 매커니즘은 **네비게이션 목적지를 Enum을 활용해 값(Value)으로 표현**하는 것이었습니다.

```swift
public enum Destination: Hashable {
  case tab(_ destination: TabDestination)
  case push(_ destination: PushDestination)
  case sheet(_ destination: SheetDestination)
  case fullScreen(_ destination: FullScreenDestination)
}

public enum TabDestination: String, Hashable {
  case home, profile, settings
}

public enum PushDestination: Hashable {
  case itemDetail(id: String)
  case comments(itemId: String)
  case replyDetail(commentId: String)
}

public enum SheetDestination: Hashable {
  case profileEdit
  case settingsDetail
}

public enum FullScreenDestination: Hashable {
  case onboarding
  case imageViewer(url: String)
}
```

#### 왜 Enum인가?

Enum을 선택한 이유는 다음과 같습니다.

- **Hashable**: NavigationPath에 저장 가능
- **Type-safe**: 잘못된 파라미터 전달 불가능
- **값 타입**: 불변성 보장, 예측 가능한 동작

### 계층적 Router 구조

각 탭과 모달이 **독립적인 Router를 가지는 트리 구조**를 설계했습니다.

```
RootRouter (level: 0)
├── HomeRouter (level: 1, tab: .home)
│   └── SheetRouter (level: 2) ← 독립적인 Router를 가져 Sheet 내에서도 화면 푸쉬 가능
├── ProfileRouter (level: 1, tab: .profile)
└── SettingsRouter (level: 1, tab: .settings)

```

```swift
// Navigation/Router.swift
@Observable
public final class Router {
  let level: Int
  weak var parent: Router?

  public var navigationStackPath: [PushDestination] = []
  public var presentingSheet: SheetDestination?
  public var presentingFullScreen: FullScreenDestination?
  public var selectedTab: TabDestination? // level 0에서만 사용

  public func childRouter(for tab: TabDestination? = nil) -> Router {
    let router = Router(level: level + 1)
    router.parent = self  // 부모 참조 유지
    return router
  }
}
```

#### 이벤트 전파 구조: 깊은 스택에서의 탭 전환

깊은 네비게이션 스택에서도 **탭 전환**이 가능하도록 이벤트를 부모로 전파합니다.

```swift
// Navigation/Router.swift
public func select(tab destination: TabDestination) {
  if level == 0 {
    // 루트 라우터: 직접 처리
    selectedTab = destination
  } else {
    // 자식 라우터: 부모로 전파
    parent?.select(tab: destination)
    resetContent()  // 자신의 상태는 초기화
  }
}
```

**플로우 예시:**
```
ReplyDetailView (level 3)
  → "Profile 탭으로 이동" 버튼 클릭
  → router.select(tab: .profile)
     → CommentsRouter (level 2)
        → ItemDetailRouter (level 2)
           → HomeRouter (level 1)
              → RootRouter (level 0)
                 → selectedTab = .profile ✅
```

### NavigationContainer: View

`NavigationStack`을 래핑하여 **Sheet/FullScreen 관리를 통합**했습니다.
이를 통해 sheet나 fullScreen을 커스텀 또는 모디파이어 추가 시에도 용이합니다.

```swift
// Navigation/NavigationContainer.swift
public struct NavigationContainer<Content: View>: View {
  @State var router: Router
  let content: Content

  public init(
    parentRouter: Router? = nil,
    tab: TabDestination? = nil,
    @ViewBuilder content: () -> Content
  ) {
    self._router = State(
      wrappedValue: parentRouter?.childRouter(for: tab) ?? Router(level: 0)
    )
    self.content = content()
  }

  public var body: some View {
    InnerContainer(router: router, content: content)
      .environment(router)
      .onAppear { router.setActive() }
      .onDisappear { router.setInactive() }
  }
}

private struct InnerContainer<Content: View>: View {
  @Bindable var router: Router
  let content: Content

  var body: some View {
    NavigationStack(path: $router.navigationStackPath) {
      content
        .navigationDestination(for: PushDestination.self) { $0.view }
        .sheet(item: $router.presentingSheet) { destination in
          NavigationContainer(parentRouter: router) {
            destination.view
          }
        }
        .fullScreenCover(item: $router.presentingFullScreen) { destination in
          NavigationContainer(parentRouter: router) {
            destination.view
          }
        }
    }
  }
}
```

#### 모달 내부의 독립적인 네비게이션

Sheet나 FullScreen 안에서도 **새로운 NavigationContainer**를 생성하여 독립적인 네비게이션을 지원하도록 했습니다.

```swift
.sheet(item: $router.presentingSheet) { destination in
  NavigationContainer(parentRouter: router) {  // 새 자식 Router 생성
    destination.view
  }
}
```

### Destination - View Mapping

Destination Enum에서 View로의 매핑을 **computed property**로 구현했습니다.

```swift
public extension PushDestination {
  @ViewBuilder
  var view: some View {
    switch self {
    case let .itemDetail(id):
      ItemDetailView(itemId: id)

    case let .comments(itemId):
      CommentsView(itemId: itemId)

    case let .replyDetail(commentId):
      ReplyDetailView(commentId: commentId)
    }
  }
}
```

#### 장점

- **컴파일 타임 보장**: 모든 케이스를 switch로 처리해야 함
- **타입 안전성**: 파라미터 타입이 Enum에 정의됨
- **중앙화**: 모든 View 매핑이 한 곳에 모임

```swift
// NavigationStack에서 자동으로 뷰 생성
.navigationDestination(for: PushDestination.self) { destination in
  destination.view  // ← Extension에서 정의한 computed property
}
```

## ✨ 달성한 결과

이러한 Destination 기반 구조를 통해 아래와 같은 결과를 얻을 수 있었습니다.

### 런타임 에러 방지

```swift
// ✅ 컴파일 타임에 모든 목적지가 보장됨
router.push(.itemDetail(id: "123"))
// → 반드시 ItemDetailView가 표시됨
```

### 보일러플레이트 대폭 감소

```swift
// ❌ Before: 이동가능한 화면이 많을 수록 보일러 플레이트 증가
class ItemDetailViewModel: NavigatableViewModel {
  func navigateToComment() {
    let viewModel = CommentViewModel()
    viewModel.setNavigator(navigator)
    navigator?.navigate(viewModel)
  }
  func navigateToSomeView() { ... }
  func navigateToAnotherView() { ... }
}
let viewModel = ItemDetailViewModel()
viewModel.navigateToComment() // 화면 전환

// ✅ After: Enum에 등록
enum PushDestination {
  case comment // 필요한 화면 케이스 추가
  
  var view: some View {
	switch self {
	case .comment:
	  CommentView()
	}
  }
}
router.push(.comment)
```

### 모달 내부 독립적인 네비게이션

```swift
ProfileView
  ↓ router.present(sheet: .profileEdit)
ProfileEditSheet (새로운 NavigationContainer 생성)
  ↓ 이 안에서도 push 네비게이션 가능
  ✅ 모달을 닫으면 자동으로 스택 정리됨
```

### 깊은 스택에서의 탭 전환

```swift
// ReplyDetailView (Home 탭의 3단계 깊이)
router.select(tab: .profile)

// → 자동으로 부모 → 부모 → 루트로 전파
// → Profile 탭으로 전환
// → Home 탭의 스택은 자동으로 초기화됨
```

### 테스트 가능한 아키텍처

```swift
// Router를 통해 네비게이션 테스트 가능
@Test("1단 푸쉬 테스트")
func test1() async throws {
    let router = Router(level: 0)
    router.push(.itemDetail(id: "123"))

    #expect(router.navigationStackPath.count == 1)
    #expect(router.navigationStackPath[0] == .itemDetail(id: "123"))
}

@Test("2단 푸쉬 테스트")
func test2() async throws {
    let router = Router(level: 0)
    router.push(.itemDetail(id: "123"))
	router.push(.comment)
	
    #expect(router.navigationStackPath == [.itemDetail(id: "123"), .comment])
}

@Test("탭 전환 테스트")
func test3() async throws {
    let rootRouter = Router(level: 0)
    let childRouter = rootRouter.childRouter(for: .home)

    childRouter.select(tab: .profile)

    #expect(rootRouter.selectedTab == .profile)
}
```

## ⚖️ 현재 아키텍쳐의 문제점

### 모듈 의존성 문제

이 아키텍처의 한계는 **View Mapping 코드가 모든 피처 모듈에 의존**해야 한다는 점입니다.
이로 인해Navigation 모듈을 만든다고 가정했을 때, 피처 모듈과의 결합도가 높아지게 되며 **순환 의존성 문제**나 **테스트에 어려움**, **빌드 시간 증가** 등의 문제가 발생할 수 있습니다.

```swift
// Destination-ViewMapping.swift
import SwiftUI

// ❌ 문제: 모든 피처 모듈을 import 해야 함
import HomeFeature
import ProfileFeature
import SettingsFeature
// ... 계속 추가됨

public extension PushDestination {
  @ViewBuilder
  var view: some View {
    switch self {
    case let .itemDetail(id):
      ItemDetailView(itemId: id)  // HomeFeature에 의존
    case let .comments(itemId):
      CommentsView(itemId: itemId)  // CommentsFeature에 의존
    // ...
    }
  }
}
```

이러한 문제를 해결하기 위해 protocol 기반 ViewFactory 패턴 등 추상화를 고려해보고 있지만, 컴파일 타입 안정성이 떨어질 수 있어 트레이드 오프를 고려해 더 나은 방식을 고민중입니다.

## 참고

- [@Observable (iOS 17+) - Apple Documentation](https://developer.apple.com/documentation/observation)
- [NavigationStack - Apple Documentation](https://developer.apple.com/documentation/swiftui/navigationstack)
- [https://github.com/fespinoza/Youtube-SampleProjects/tree/main](https://github.com/fespinoza/Youtube-SampleProjects/tree/main)