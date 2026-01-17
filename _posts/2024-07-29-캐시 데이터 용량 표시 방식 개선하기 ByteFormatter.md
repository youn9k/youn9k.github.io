---
title: "[Swift]캐시 데이터 용량 표시 방식 개선하기 ByteFormatter"
date: 2024-07-29 12:00 +0900
categories:
  - 🍎 iOS
  - Swift
tags:
  - ByteFormatter
  - 포맷팅
---

## 캐시 데이터 용량 표시 방식 개선하기: ByteFormatter


프로젝트를 진행하며 이미지 캐시를 제거하는 기능을 개발하게 되었는데요.
캐시 데이터 용량을 어떻게 표기할까 고민하다 너무 작지도 크지도 않도록 `MB` 단위로 표현하기로 정하고 byte를 1024 로 2번 나누어 MB로 변환하여 표기했습니다.

하지만 코드리뷰를 진행하며 팀원분께 더 나은 방식을 배웠는데요.
![](/assets/img/post/2024/002.png)

까먹지 않도록 정리해두려고 합니다!


## 왜 ByteFormatter를 사용해야 할까?

기존 방식에서는 용량을 항상 MB 단위로 표시했습니다. 이렇게 되면 `GB` 이상의 큰 용량을 표현하거나, `KB`나 `Byte` 단위의 작은 용량일 때는 어색해 보일 수 있습니다. 이를 개선하기 위해 `ByteFormatter`를 사용하면 Byte, KB, MB, GB 등 적절한 단위로 가변적으로 표시할 수 있어 매우 유용합니다.

아래는 캐시 데이터 용량을 MB 단위로 고정하여 표시하던 기존 코드입니다.

```swift
ImageCache.default.calculateDiskStorageSize { result in
    switch result {
    case let .success(size):
        let mbSize = (Double(size) / 1024 / 1024)
        let sizeString = String(format: "%.2f MB", mbSize)
		print(sizeString) // 10.23 MB
    case let .failure(error):
        let description = error.localizedDescription
        print(description)
    }
}
```

## ByteFormatter를 활용한 개선된 코드

이제 `ByteFormatter`를 사용하여 용량을 더 직관적으로 표시하도록 개선해보겠습니다.

```swift
ImageCache.default.calculateDiskStorageSize { result in
    switch result {
    case let .success(size):
        let formatter = ByteCountFormatter()
        formatter.allowedUnits = .useAll // 모든 단위 사용 가능
        formatter.countStyle = .decimal // 소수점 단위로 표기
        
        // 크기를 ByteFormatter를 통해 문자열로 변환합니다.
        let sizeString = formatter.string(fromByteCount: Int64(size))
		print(sizeString) // 45 KB, 10 MB, 1 GB, ...
    case let .failure(error):
        let description = error.localizedDescription
        print(description)
    }
}
```

- `ByteCountFormatter`: 용량을 적절한 단위로 표시할 수 있게 도와주는 클래스입니다.
- `allowedUnits`: 사용할 수 있는 단위를 지정합니다. `.useAll`을 사용할 경우 size에 맞게 Byte, KB, MB, GB 등을 자동으로 선택합니다.
- `countStyle`: `decimal` 옵션을 사용해 소수점 단위로 표기할 수 있습니다. 항상 소숫점이 표기되는 것은 아니며 내부 로직에 의해 소숫점이 필요하다고 판단될 때 추가된다고 합니다.

이제 캐시 데이터 용량과 상관없이 사용자에게 더 직관적인 용량 정보를 제공할 수 있게 되었습니다!

## 결과
![](/assets/img/post/2024/003.png)