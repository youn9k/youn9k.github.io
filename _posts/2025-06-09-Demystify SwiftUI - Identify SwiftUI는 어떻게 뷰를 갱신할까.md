---
title: "[SwiftUI]Demystify SwiftUI - Identify SwiftUI는 어떻게 뷰를 갱신할까"
date: 2025-06-09 12:00 +0900
categories:
  - 🍎 iOS
  - SwiftUI
tags:
  - SwiftUI
  - WWDC
---

## SwiftUI는 어떻게 뷰를 갱신할까?

![](/assets/img/post/2025/058.png)

SwiftUI는 이전 뷰의 상태와 현재 뷰의 상태를 비교하여 재렌더링이 필요한 뷰만 다시 그린다고 알고 있습니다. 그렇다면 어떻게 이전 뷰의 상태와 현재 뷰의 상태를 비교하는걸까요?

## SwiftUI는 현재 뷰의 상태가 이전 뷰의 상태와 달라졌는지를 어떻게 구분할까?


![](/assets/img/post/2025/059.png)

![](/assets/img/post/2025/060.png)

SwiftUI는 정체성(Identity)을 통해 뷰를 구분합니다.

UIView, NSView를 클래스로 모델링한 UIKit, AppKit과 달리 SwiftUI에선 View를 Struct로 취급하기 때문에 View를 식별하기 위해선 별도의 ID가 필요합니다.

정체성은 명시적 정체성, 구조적 정체성 두가지 방식으로 표현될 수 있으며, 명시적 정체성을 부여하지 않더라도 뷰 계층 구조를 활용해 뷰에 암시적인 ID를 부여합니다.

### 명시적 정체성 (Explicit Identity)

개발자가 ID 나 Identifiable 프로토콜을 사용하여 뷰나 데이터에 직접 고유한 식별자를 제공하는 방식입니다. ForEach와 같은 데이터 기반 컴포넌트에서 특히 중요하며, SwiftUI가 컬렉션의 항목을 효율적으로 식별하고 변경 사항에 따라 애니메이션을 적용할 수 있도록 합니다. UIKit의 포인터 정체성과 달리 SwiftUI 뷰는 값 타입이므로 별도의 명시적 식별자가 필요합니다.

### 구조적 정체성 (Structural Identity)

구조적 정체성(Structural identity)은 SwiftUI가 뷰의 **유형(type)과 뷰 계층 구조 내에서의 위치(position)** 를 기반으로 암시적인 정체성을 생성하는 방식입니다. 

```swift
```swift
@available(iOS 13.0, macOS 10.15, tvOS 13.0, watchOS 6.0, *)
extension ViewBuilder {

    public static func buildIf<Content>(_ content: Content?) -> Content? where Content : View

    public static func buildEither<TrueContent, FalseContent>(first: TrueContent) -> _ConditionalContent<TrueContent, FalseContent> where TrueContent : View, FalseContent : View

    public static func buildEither<TrueContent, FalseContent>(second: FalseContent) -> _ConditionalContent<TrueContent, FalseContent> where TrueContent : View, FalseContent : View
}
```

예를 들어, if 문과 같은 조건부 로직은 _ConditionalContent 라는 제네릭 뷰로 변환되며, 이 뷰는 참(true) 및 거짓(false) 내용에 대한 제네릭을 가집니다. 이러한 변환은 ViewBuilder에 의해 이루어지며, 이를 통해 개발자가 명시적으로 식별자를 지정하지 않아도 SwiftUI가 뷰를 구별할 수 있도록 합니다.


## Identify 를 사용할 때 주의할 점

### 상태 초기화

시간이 지남에 따라 뷰의 상태(state)가 변경되어도 뷰의 정체성(Identify)이 동일하다면, SwiftUI는 이를 동일한 뷰로 간주합니다. 

또한 뷰의 상태가 변경되더라도, 정체성은 유지됩니다.

하지만 View의 수명(Lifetime)은 정체성과 일치합니다. 정체성이 변경되면 다른 View로 간주되고, 초기화 됩니다. 이 때 해당 View가 갖고있던 상태 또한 초기화되기 때문에 주의해야 합니다.

### 애니메이션이 예상과 다를 수 있음

동일한 뷰로 인식되어 부드러운 애니메이션 전환이 일어나야 할 곳에서 갑자기 사라지고 나타나는(fade in/out) 현상이 발생할 수 있습니다.
