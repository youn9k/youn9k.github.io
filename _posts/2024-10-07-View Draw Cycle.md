---
title: "[UIKit]View Draw Cycle"
date: 2024-10-07 12:00 +0900
categories:
  - 🍎 iOS
  - UIKit
tags:
  - UIKit
  - View Draw Cycle
---

---
"\bdeadline": 
finish: true
tags:
  - UIView
  - RunLoop
  - layoutSubviews
---
오늘은 UIView Draw Cycle에 대해 정리해볼거에요.
평소에 layoutSubviews와 같은 UIView Draw Cycle에 해당하는 메소드를 오버라이딩하여 사용하는 경우가 있었는데, 이 메소드가 언제 호출되는지도 모르고 사용하는 건 잘못되었다고 생각하여 이번 기회에 정리해보려고 합니다!

틀린 내용이 있을 수 있습니다. 😂

## Run Loop

View Draw Cycle을 알아보기 전에 먼저 런루프에 대한 사전지식이 필요합니다.

하지만 아직 런루프에 대해 학습하기 전이고,
오늘은 뷰 드로잉 사이클에 대해 학습해보기로 했으므로 간단하게 알아보겠습니다.

런루프란?
- **Event(Input source, timer)를 처리하는 루프 객체**
- 수행해야 하는 작업이 있을 때 thread를 일하게 하고, 작업이 없을 때 thread를 쉬게하려는 목적으로 고안됨
- 모든 스레드마다 하나씩 존재함
- 자동으로 시작되지 않음
- **메인스레드**에서만 자동으로 실행

iOS 에서 발생하는 모든 이벤트는 RunLoop를 거쳐 처리됩니다.
RunLoop는 모든 스레드에 하나씩 존재하면서 시스템에 들어오는 두 종류의 이벤트 Input Source 와 Timer Source를 처리하는 객체입니다.

UIApplication이 실행시켜주는 메인 스레드를 제외하고는 런루프는 자동으로 실행되지 않습니다.

![](/assets/img/post/2024/049.png)

위 그림은 메인 스레드의 런루프인 메인 런루프의 실행 순서입니다.

터치와 같은 유저이벤트가 발생하면 이벤트(시스템 영역)를 생성, 시스템에 들어오는 이벤트들은 포트를 거쳐 Event Queue라는 대기열에 쌓이게 되고, 런루프는 실행이 될 때 여기에 있는 이벤트들을 가져와 처리하게 됩니다.
(루프라고 해서 자체적으로 무한 루프처럼 동작할 것 같지만 그렇지 않음)

런루프가 가져온 이벤트들은 UIApplication 객체에 전달되고 Core Object들 안에 있는 핸들러를 호출해줍니다.

핸들러들은 우리가 작성한 코드를 호출해주는데, 이러한 메소드들이 반환되면 다시 제어는 메인 런루프로 돌아가게 됩니다.
> 이 부분이 이해가 잘 안되는데, 자료를 찾기 어렵네요ㅜㅜ

그 다음 메인 런루프는 View들을 배치하고 다시 그리는 역할을 하는 **Update Cycle**을 수행합니다.

![](/assets/img/post/2024/050.png)

## Update Cycle

Update Cycle은 Constraint, Layout, display 작업의 사이클을 뜻합니다.

이러한 사이클은 런루프의 마지막 단계에서 실행되기 때문에, 
이벤트의 발생과 View가 갱신되기 까지는 **딜레이**가 존재합니다.

> iOS 애플리케이션은 초당 60~120 사이클이 실행되기 때문에, 유저는 UI와 상호작용간의 차이를 느끼지 못합니다. (1프레임당 1사이클)

그렇기에 내가 View를 갱신하기를 원하는 시점과 실제로 갱신되는 시점이 다를 수 있습니다.

이러한 문제를 해결하기 위해 애플에선 View의 업데이트 메소드를 제공합니다.

Update Cycle 은 위에 언급했듯이, 아래의 3단계를 거쳐 View를 갱신합니다.

1. constraint: 오토레이아웃 제약 조건을 계산합니다.
2. layout: 제약 조건에 맞추어 layout을 계산합니다.
3. display: 계산된 layout대로 화면에 배치합니다. 

## Constraint

### updateConstraints()

오토레이아웃을 사용하는 뷰의 constraint 들을 동적으로 바꾸는 메소드.
updateConstraint는 오직 오버라이딩되어야 하며 명시적으로 호출되어서는 안됨.
constraint 를 업데이트하고 싶다면, 시스템에게 updateConstraint()를 호출하도록 요청해야함.

### setNeedsUpdateConstraints()
다음 Update Cycle에서 제약조건들에 대한 갱신이 일어나도록 `updateConstraints()` 를 예약하는 메소드

### updateConstraintsIfNeeded()
오토레이아웃을 사용하는 뷰에 대해서 즉시 `updateConstraints()` 호출을 시스템에게 요청하고 런루프가 끝날 때까지 기다리지 않는 메소드

## Layout


### layoutSubviews()

UIView와 하위 모든 서브뷰들에 대한 **위치와 크기의 변경**을 처리하는 메소드.
제약 조건을 통해 각 뷰들의 레이아웃을 계산한다.
이 메소드는 해당 뷰와 모든 하위뷰에 대해 적용되기 때문에 비용이 비싼 메소드입니다.
그렇기에 애플에선 오버라이딩하는 것 이외에 외부에서 호출하지 말라고 하고 있습니다.
대신 아래에서 소개할 메소드들을 통해 간접적으로 호출할 수 있습니다.

### setNeedsLayout()

이 메소드는 시스템에게 자신을 호출한 뷰의 레이아웃이 다시 계산되어야 한다고 알려주고 즉시 반환됩니다.
대신 뷰는 **다음 update cycle에 갱신됩니다.**


### layoutIfNeeded()

이 메소드는 setNeedsLayout과 같이 간접적으로 layoutSubviews를 호출할 수 있는 메소드입니다.
하지만 다음 update cycle에 예약하는 것이 아니라 **즉시 레이아웃 업데이트를 요청합니다.**

> 호출한다고 무조건 업데이트해주는 것은 아니고, 변경사항이 있다고 마킹된 뷰들에 대해서만 업데이트해준다고 합니다.

## Display

### draw(_: )

이 메소드는 호출한 뷰의 디스플레이를 그리는 메소드입니다.
서브뷰들에 연쇄적으로 호출되던 layoutSubviews와 다르게 이 메소드는 **호출한 뷰의 디스플레이만 다시 그립니다**.
이 메소드 또한 시스템이 호출해주는 메소드이므로 외부에서 호출하지 않습니다.
대신 아래에서 소개하는 메소드들을 통해 간접적으로 호출할 수 있습니다.

### setNeedsDisplay()

동작 방식은 setNeedLayout() 이랑 같습니다.
디스플레이에 변경사항이 있으니 draw()를 호출해달라고 시스템에게 예약합니다.
그리고 다음 사이클에서 다시 그리게됩니다.

뷰의 특정 부분만 업데이트하고 싶다면 CGRect를 인자로 줄 수도 있습니다.

![](/assets/img/post/2024/051.png)


### displayIfNeeded()

이 메소드 또한 동작방식은 layoutIfNeeded() 와 같습니다.
시스템에게 즉시 draw() 를 호출해달라고 요청합니다.

하지만 런루프의 흐름을 깨뜨리는건 좋지 않은 방법이고, 
공식문서에서도 setNeedDisplay()를 사용하기를 귄장하고 있습니다

![](/assets/img/post/2024/052.png)

## 정리!

위 내용들을 표로 정리하면 아래와 같이 됩니다!

![](/assets/img/post/2024/053.png)

## UIViewController & UIView 라이프사이클

아래 그림은 ViewController의 라이프사이클과 UIView 가 그려지는 순서입니다.
위에서 설명한 constraint -> layout -> display phase 순서대로 진행되는걸 볼 수 있습니다.

drawRect 와 draw 는 둘다 UIView의 메소드인데, 차이점은 잘 모르겠네요ㅜㅜ
알게 된다면 추후 업뎃을 해보겠습니다.

![](/assets/img/post/2024/054.png)

## 서브뷰 구조에서의 호출 순서


![](/assets/img/post/2024/055.png)

> wwdc18 - High Performance Auto Layout

update Constraint는 가장 위에 있는 서브뷰부터 계산되고, layout은 베이스뷰부터 계산된다.
display 도 우리 눈에 보이듯이 베이스뷰부터 그려지게 된다.