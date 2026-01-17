---
title: "[iOS]iOS Hang, Hitch 그리고 Render Loop"
date: 2025-06-16 12:00 +0900
categories:
  - 🍎 iOS
  - Swift
tags:
  - 성능
  - Hang
  - Hitch
---

앱이 멈추거나 끊기는 이유는 크게 2가지로 나뉩니다.
Hang(앱이 반응하지 않는 문제)과 Hitch(스크롤이 버벅이는 프레임 드롭 문제)

## Hang과 Hitch 구분

### Hang: 메인 스레드의 멈춤 현상
앱 전체가 멈추고 터치 등 사용자 입력에 전혀 반응하지 않는 현상으로, 메인 스레드 내 작업이 너무 많을 경우 발생합니다.

![](/assets/img/post/2025/065.png)

Hang은 2가지 시나리오로 발생할 수 있는데, 메인스레드가 너무 많은 작업을 수행하는 경우와 단일 긴 작업으로 인해 블로킹되는 경우입니다.


![](/assets/img/post/2025/066.png)

예를 들어 화면에 보여지는 이미지는 4개지만, 전체 이미지를 준비하려고 할 경우 아래와 같이 메인스레드가 불필요하게 너무 많은 작업을 수행하게 되고 Hang으로 이어집니다.

그리고 메인 스레드에서 네트워크 요청을 동기로 수행한다면, 네트워크 응답이 올 때까지 무기한으로 기다리게 되고 메인스레드가 블로킹되어 Hang으로 이어지게 되므로 네트워크 작업과 같이 오래걸리는 작업은 비동기로 요청하거나 다른 스레드에서 작업하도록 해야 합니다.
![](/assets/img/post/2025/067.png)

### Hitch: 렌더 루프에서의 끊김 현상

Hitch는 렌더 루프(Render Loop)에서 각 프레임의 생성이 마감 시간(VSYNC) 내에 완료되지 못했을 때 발생합니다.구형 아이폰이나 아이패드와 같이 60fps의 주사율을 가진 기기는 매 프레임이 16.67 ms 마다 보여지고, 120fps의 주사율을 가진 최신 기기들은 8.8.3 ms 마다 보여지게 됩니다. 이 시간이 조금만 넘겨도 프레임이 누락되며 스크롤이나 애니메이션이 끊기는 현상이 발생합니다.

## 렌더 루프의 이해

렌더 루프는 터치 이벤트를 앱에 전달하고 UI 변경 사항을 OS로 전달하여 화면에 그려질 프레임을 확정하는 연속적인 프로세스를 말합니다. 이 루프는 장치의 주사율에 따라 동작합니다.

## VSYNC: 프레임 준비 시간

시스템은 프레임이 교체되는 시점을 알리기 위해 VSYNC라는 이벤트를 발생시킵니다.
렌더 루프는 VSYNC에 시간이 맞춰져있으며 프레임을 준비하기 위해선 3단계의 체크포인트를 거쳐야 합니다.

![](/assets/img/post/2025/068.png)

##  렌더 루프 3 스테이지

앱 스테이지: 앱 내에서 이벤트가 처리되고 UI를 변경하는 단계

렌더 서버 스테이지: UI가 GPU에서 렌더링되는 단계

디스플레이 스테이지: 렌더링된 프레임이 실제로 화면에 보여지는 단계

![](/assets/img/post/2025/069.png)

## 더블 버퍼링 & 트리플 버퍼링

각 스테이지는 순차적으로 진행되기 때문에, VSYNC안에 모든 작업을 마무리하지 못하면 Hitch로 이어지기 때문에 2 프레임 전에 시작, 3 프레임 전에 시작하는 기법을 각각 Double Buffering, Triple Buffering이라고 부릅니다.

![](/assets/img/post/2025/070.png)
![](/assets/img/post/2025/071.png)


## 렌더 루프 5단계

렌더 루프는 더 자세히 5단계로 구분할 수 있습니다.

![](/assets/img/post/2025/072.png)

### Event Phase

이벤트 단계는 앱이 터치, 네트워크 콜백, 키보드 입력, 타이머 등 이벤트를 처리하고 UI 변경 필요 여부를 결정하는 단계입니다. `setBackgoundColor()` 등 UI를 변경하고, Layout이나 Display 변경이 필요한 뷰들을 식별합니다.

Layer의 bounds가 변경된 경우, Core Animation은 자체적으로 setNeedsLayout을 호출하여 레이아웃을 다시 계산해야하는 모든 뷰를 식별합니다.

![](/assets/img/post/2025/073.png)

### Commit Phase

커밋 단계는 레이아웃과 드로우를 담당하는 단계입니다. 

Layout이 필요한 뷰들을 부모에서 자식 순으로 배치하여 Layer Tree를 생성합니다. 그 다음 drawRect를 override한 커스텀 View나 setNeedsDisplay가 호출된 View들이 그려집니다.
모두 그려지게 되면 Layer Tree를 렌더링 하기 위해 렌더 서버로 제출합니다.

![](/assets/img/post/2025/074.png)

### Render Prepare Phase

렌더 준비 단계는 모든 Layer Tree를 순회하여 GPU가 실행할 수 있는 선형 파이프라인을 준비합니다.
파이프라인은 부모 뷰에서 자식 뷰 순으로 뒤 -> 앞으로 배치됩니다.

![](/assets/img/post/2025/075.png)
![](/assets/img/post/2025/076.png)

### Render Execute Phase

렌더 실행 단계는 전달된 선형 파이프라인을 통해 GPU가 최종 Texture를 생성합니다.
"일부 레이어는 렌더링하는데 더 오래 걸릴 수 있으며, 흔한 렌더 병목 사유입니다."

![](/assets/img/post/2025/077.png)

### Display Phase

GPU가 프레임을 렌더링하면 다음 VSYNC에서 사용자 디스플레이에 표시될 준비를 합니다.

## 병렬 렌더 루프

실제 프로세스는 평균 프레임 률을 높이기 위해 각 렌더 루프 스테이지를 병렬적으로 실행합니다.
앱 스테이지, 렌더 서버 스테이지, 디스플레이 스테이지를 동시에 수행하며 프레임을 병렬적으로 준비합니다.

![](/assets/img/post/2025/078.png)


## Hitch 종류

이러한 Hitch는 크게 2가지로 분류합니다.

![](/assets/img/post/2025/079.png)

### Commit Hitch

앱 프로세스 내에서 발생하는 HItch로 **"앱이 이벤트를 처리하거나 Commit하는 데 너무 오래 걸릴 때"** 발생합니다. 커밋이 VSYNC의 마감 시간을 넘어가면 해당 VSYNC 동안 렌더 서버는 처리할 Layer Tree가 없어 직전 프레임이 그대로 보여집니다.

앱 프로세스 내에서 발생하는 Hitch이기 때문에 개선할 여지가 많습니다.
불필요한 layoutIfNeeded() 를 제거하거나 draw 함수를 override하여 무거운 작업을 수행하지 않도록 할 수 있습니다.

```swift
// view.layoutIfNeeded()
view.setNeedsLayout()
```

### Render Hitch

렌더 서버에서 발생하는 Hitch로 **"렌더 서버가 레이어 트리를 처리하는데 너무 오래 걸릴 때"** 발생합니다. 렌더 실행 단계가 너무 오래 걸려 VSYNC의 마감 시간을 넘어가면 프레임이 준비되지 않아, 직전 프레임이 보여지며 한 프레임 늦게 표시되게 됩니다.

앱 프로세스 내에서 발생하는 Hitch가 아니기 때문에 개선할 여지가 적지만, 그림자나 마스크 처리 등 GPU에 부하를 주는 코드를 제거해 개선할 수 있습니다.

```swift
view.layer.shadowOpacity = 1
```

## 렌더 루프 5단계 및 Hitch 가능성 요약


| 단계                 | 설명                                           | Hitch 가능성 |
|:-------------------- |:---------------------------------------------- |:------------:|
| Event Phase          | 사용자 이벤트, 네트워크 콜백, 타이머 등 처리   | Commit Hitch |
| Commit Phase         | 레이아웃, Draw 수행 후 Layer Tree 생성 및 전송 | Commit Hitch |
| Render Prepare Phase | 렌더 서버가 트리 순회 및 렌더 파이프라인 준비  | Render Hitch |
| Render Execute Phase | GPU에서 Texture 렌더링                         | Render Hitch |
| Display Phase        | 완성된 프레임 화면에 출력                      | ?(기기 이슈) |


## 정리

실제로는 Hang과 Hitch가 복합적으로 발생하는 경우가 많습니다.
메인 스레드에서 Hang이 발생하면 자연스레 Main Run Loop가 지연되고,
Commit Phase가 지연되어 Commit Hitch가 발생합니다.
따라서 터치와 같은 사용자 이벤트도 먹통되며 프레임이 떨어져 화면이 끊겨 보이는 현상이 발생하여 사용자 경험에 악영향을 주게 됩니다.

쾌적한 사용자 경험을 제공하기 위해 메인 스레드를 Block 하거나 바쁘지 않게 만들고, 렌더 루프에서 HItch가 발생하지 않도록 신경써야 합니다.

또한 이를 개선할 수 있도록, XCode 디버그 콘솔 창에도 기본적으로 500ms 이상의 Hang에 대해 경고문을 띄워주고 있습니다. 더 자세한 로그를 확인하려면 Instruments를 통해 앱의 Behind를 보고 문제를 해결할 수 있습니다.


## 참고

https://developer.apple.com/videos/play/wwdc2021/10258/

https://developer.apple.com/kr/videos/play/tech-talks/10855

