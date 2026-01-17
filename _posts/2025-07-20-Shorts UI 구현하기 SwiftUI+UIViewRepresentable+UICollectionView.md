---
title: "[SwiftUI]Shorts UI 구현하기 SwiftUI+UIViewRepresentable+UICollectionView"
date: 2025-07-20 12:00 +0900
categories:
  - 🍎 iOS
  - SwiftUI
tags:
  - SwiftUI
  - UIKit
  - 하이브리드
---

## 배경

유튜브 Shorts, 인스타그램 릴스, TikTok과 같은 숏폼 콘텐츠 UI를 어떻게 구현할 수 있을까요?
최근 사이드 프로젝트를 진행하며 Shorts 화면을 담당하게 되었고, 구현하며 마주한 고민들과 기술적인 선택, 그리고 그 안에서 마주했던 문제 해결 경험을 정리해보려고 합니다.

## 요구사항 정의

구현에 들어가기 전 먼저 만족해야 하는 요구사항을 아래와 같이 정리할 수 있었습니다.
기능 명세는 일반적인 숏폼 컨텐츠를 떠올리며 정의할 수 있었고, 기존 SwiftUI 기반의 앱 구조에 안정적으로 통합되기를 원했습니다.

- 한번에 하나의 영상이 전체화면을 채우는 UI 
- 세로 방향 스크롤을 통해 영상 전환 및 페이징 UX
- 스크롤 시 영상 전환이 빠르고 부드러울 것
- 좋아요, 댓글, 사용자 정보 등 오버레이가 가능할 것
- SwiftUI 코드 환경과 통합이 가능할 것

## 유사 서비스 분석

숏폼 콘텐츠 UI를 구현해보는 게 처음이었기에, 유사한 서비스들이 어떤 방식으로 동작하는지 먼저 분석해보았습니다. 그중 요구된 디자인과 가장 유사한 YouTube Shorts를 기준으로 동작 방식을 살펴보았습니다.

주의 깊게 살펴본 부분은 '어떻게 빠르고 자연스러운 영상 전환을 구현했는지' 였습니다.
여기서 인상적이었던 점은, 스크롤이 멈춘 뒤 영상이 재생되기까지 로딩 시간이 필요하기 때문에, 이를 고려해 유튜브는 처음엔 영상의 썸네일 이미지를 보여주고, 로딩이 완료되면 실제 영상 화면으로 자연스럽게 전환하는 방식을 사용하고 있었습니다.


|                                       |                                      |
| ------------------------------------- | ------------------------------------ |
| ![](/assets/img/post/2025/101.jpeg) | ![](/assets/img/post/2025/102.gif) |


이를 참고하여, 썸네일 이미지를 먼저 노출하고 영상 로딩이 완료되면 AVPlayerLayer를 썸네일 이미지 Layer 위에 추가하는 방식으로 성능을 고려한 자연스러운 재생 방식을 구현할 수 있겠다 생각했습니다.

## 어떤 기술을 쓸까? LazyVStack vs List vs UICollectionView

SwiftUI에서 세로로 반복되는 UI를 보여주기 위해 가장 먼저 LazyVStack과 List를 떠올렸습니다.
하지만 페이징 UX 적용, 영상 사전 로딩(Pre-Patching), 셀 재사용을 통한 메모리 최적화를 고려했을 때 UICollectionView + UIViewRepresentable 방식을 채택했습니다.

## 뷰 구조 설명 및 실제 화면

설계한 UI 구조는 다음과 같은 계층 구조로 이루어집니다.

```
ShortsView(SwiftUI)
    - TabView(추천 / 팔로잉 / 구독)
            - ShortsContentView (UIViewRepresentable)
                - ShortsContentCollectionView (컬렉션뷰)
                    - ShortsContentCell (컬렉션뷰 셀)
            - ShortsContentView (UIViewRepresentable)
                - ShortsContentCollectionView (컬렉션뷰)
                    - ShortsContentCell (컬렉션뷰 셀)
            ...
    - ShortsOverlayView (상단 필터 및 검색 버튼)
```

SwiftUI로 구성된 ShortsView는 최상위 뷰로, 좌우 스와이프를 통해 세가지 필터(추천, 팔로잉, 구독)을 변경할 수 있습니다. 각 탭에는 ShortsContentView가 들어가는데, 이는 UICollectionView를 SwiftUI에서 사용할 수 있도록 UIViewRepresentable를 채택하여 래핑한 커스텀 뷰입니다.

이렇게 구성된 화면은 아래와 같습니다.

![](/assets/img/post/2025/103.png)

## SwiftUI와 UICollectionView 연동하기 - UIViewRepresentable + Coordinator

### UIVIewRepresentable

SwiftUI에서 UIKit 컴포넌트를 사용하려면 UIViewRepresentable을 사용해야 합니다.

> UIViewRepresentable을 채택하면 makeUIView와 updateUIView를 구현해야 하며, 두 메소드에 대해 짧게 설명하자면 아래와 같습니다.
> makeUIView는 SwiftUI 뷰가 생성될 때 호출되며 UIView 인스턴스를 생성하고 초기화하는 메소드입니다. 
> updateUIView는 SwiftUI 뷰가 업데이트되는 시점에 호출되며, State-Binding을 통해 변경된 데이터를 UIView에 업데이트할 때 사용합니다. 

```swift
import SwiftUI

struct ShortsContentView: UIViewRepresentable {
  let tag: ShortsFilter
  weak var viewModel: ShortsViewModel?

  func makeUIView(context: Context) -> ShortsContentCollectionView {
    let view = ShortsContentCollectionView()
    view.delegate = context.coordinator
    view.dataSource = context.coordinator
    context.coordinator.collectionView = view
    context.coordinator.setupInitialPlayerLayer()
    return view
  }

  func updateUIView(_ uiView: ShortsContentCollectionView, context: Context) {}

  func makeCoordinator() -> ShortsContentView.Coordinator {
    return Coordinator(
      tag: tag,
      viewModel: viewModel
    )
  }
}
```

### Coordinator

그리고 UICollectionView의 Delegate, DataSource 역할을 하기 위해 Coordinator를 정의하여 CollectionView에서 발생하는 이벤트, 데이터 업데이트 등 뷰 로직에 대한 책임을 View로부터 분리하는 구조로 작성했습니다.
> Coordinator는 SwiftUI와 UIKit 사이의 인터랙션을 담당합니다.

```swift
import Combine
import SwiftUI

extension ShortsContentView {
  /// ShortsView(SwiftUI) <-> ShortsContentCollectionView(UIKit) 사이의
  /// 뷰 로직(Delegate, DataSource, Event Handling 등)을 담당
  final class Coordinator: NSObject {
    private var cancellables = Set<AnyCancellable>()
    let tag: ShortsFilter
    weak var collectionView: ShortsContentCollectionView?
    weak var viewModel: ShortsViewModel?

    init(
      tag: ShortsFilter,
      viewModel: ShortsViewModel?
    ) {
      self.tag = tag
      self.viewModel = viewModel
      super.init()
      bindEvent()
    }
  }
}
```

가독성을 위해 기능 별로 extension을 분리해 작성했습니다.

```swift
// MARK: - UICollectionView Delegate, DataSource

extension ShortsContentView.Coordinator: UICollectionViewDataSource, UICollectionViewDelegateFlowLayout {
  func collectionView(
    _ collectionView: UICollectionView,
    cellForItemAt indexPath: IndexPath
  ) -> UICollectionViewCell {
    guard let cell = collectionView.dequeueReusableCell(
      withReuseIdentifier: ShortsContentCell.reuseIdentifier,
      for: indexPath
    ) as? ShortsContentCell else {
      return UICollectionViewCell()
    }
    cell.delegate = self
    return cell
  }

  // ...
}

// MARK: - ShortsContentCellActionDelegate

extension ShortsContentView.Coordinator: ShortsContentCellActionDelegate {
  func didTapContent(_ cell: ShortsContentCell) {
    if let index = index(for: cell) {
      viewModel?.handleAction(.tapContent(index: index))
    }
  }

  // ...
```

또한 3개의 ShortsContentView 사이에서 필터 선택 정보를 동기화하고 PlayerLayer를 옮기는 등 성능 최적화를 위해 Combine을 활용해 ViewModel과 바인딩했습니다.
```swift
private extension ShortsContentView.Coordinator {
  func bindEvent() {
    viewModel?.filterWillChange
      .filter { $0.filter == self.tag }
      .map { IndexPath(item: $0.oldIndex, section: 0) }
      .sink { [weak self] indexPath in
        if let cell = self?.collectionView?.cellForItem(at: indexPath) as? ShortsContentCell {
          cell.detachPlayerLayer()
        }
      }
      .store(in: &cancellables)

    viewModel?.filterDidChanged
      .filter { $0.filter == self.tag }
      .map { IndexPath(item: $0.newIndex, section: 0) }
      .sink { [weak self] indexPath in
        if let cell = self?.collectionView?.cellForItem(at: indexPath) as? ShortsContentCell,
           let playerLayer = self?.viewModel?.playerLayer {
          cell.attach(playerLayer: playerLayer)
        }
      }.store(in: &cancellables)
// ...
}
```

## AVPlayerLayer를 각 셀에 Attach/Detach

Shorts UI의 핵심은 빠른 영상 전환과 자연스러운 재생 경험이라고 판단했습니다.
이를 위해 초기에는 각 셀마다 AVPlayer를 개별로 생성하는 방식으로 구현했지만, 이는 메모리와 성능 측면에서 비효율적이었습니다.

이에 따라 방식을 변경해, 하나의 AVPlayer 인스턴스만 생성하고 AVPlayerLayer를 공유하는 방식으로 전환했습니다. 사용자가 스크롤을 통해 보여지는 셀을 변경할 때마다 재생할 URL을 바꿔주고, 해당 셀에 AVPlayerLayer를 attach하는 방식으로 전환했습니다.

이 방식은 AVPlayer 인스턴스를 하나만 사용하므로 메모리 사용을 최소화할 수 있고, 재생 상태 관리를 일관되게 관리할 수 있어 관리가 용이해진다는 장점이 있습니다.


```swift
// ShortsContentCell.swift
func attach(playerLayer: AVPlayerLayer) {
    playerLayer.frame = thumbnailImageView.bounds
    thumbnailImageView.layer.addSublayer(playerLayer)
  }

  func detachPlayerLayer() {
    thumbnailImageView.layer.sublayers?.removeAll(where: { $0 is AVPlayerLayer })
  }
```

하지만 AVPlayerLayer는 GPU를 통해 렌더링되기 때문에 불필요한 셀에 남아있을 경우 성능 저하를 유발할 수 있습니다. 이를 방지하기 위해 아래와 같은 레이어 교체 전략을 적용했습니다.
- `scrollViewWillEndDragging(_:)` 시점에 다음 셀에 attach
- `didEndDisplaying(_:)` 시점에 이전 셀에서 detach

이를 통해 사용자 스크롤에 맞춰 필요한 셀에만 레이어를 붙이도록 함으로써, 렌더링 리소스를 효율적으로 관리하면서 자연스러운 영상 재생 경험을 제공할 수 있게 되었습니다.

## 결과

아직 해결해야 할 과제들도 남아있습니다. 다음 영상에 대해 사전 로딩(Pre-Fetching), AVPlayerLayer를 attach할 때 레이어가 셀 크기만큼 확장되는 문제 등 개선해야할 부분이 많습니다.

그럼에도 이번 경험을 통해 요구사항을 분석해 SwiftUI와 UIKit을 적절히 섞어 활용해 볼 수 있었고, 영상 재생에 따른 성능 최적화 등 다양한 기술적 고민도 해볼 수 있었습니다.


| 세로 스크롤 시                       | 좌우 스와이프로 필터 교체 시         |
| ------------------------------------ | ------------------------------------ |
| ![](/assets/img/post/2025/104.gif) | ![](/assets/img/post/2025/105.gif) |










