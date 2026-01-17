---
title: "[Swift]GCD Sync, Async, Serial, Concurrent 조합해보기"
date: 2024-07-31 12:00 +0900
categories:
  - 🍎 iOS
  - Swift
tags:
  - GCD
  - Concurrency
---

GCD를 활용해 비동기 코드를 작성하며 자주 헷갈리는 sync, async 그리고 Serial, Concurrent를 조합했을 때 어떻게 동작하는지 그림과 함께 정리해보려고 합니다.
GCD가 무엇인지는 [이 곳](https://youngkdevlog.tistory.com/84)에서 확인 가능합니다!

## Serial + Sync
```swift
var numbers = [0, 1, 2, 3, 4]
let dispatchQueue = DispatchQueue(label: "custom")

print("Serial + Sync 실행결과")
(0..<numbers.count).forEach { index in
    dispatchQueue.sync {
        print(numbers[index])
    }
}
```
![](/assets/img/post/2024/004.png)
![](/assets/img/post/2024/005.png)


> 😨 메인스레드 데드락 주의!
> 메인 스레드에서 sync로 호출하면, 메인 스레드는 작업결과를 리턴받을 때까지 기다리게되고(block) 메인 스레드는 기다리느라 아무것도 할 수 없으므로 작업을 수행할 수 없습니다.
> 따라서, 작업은 "메인스레드야, 내 작업 해줘!" 외치고 있고, 
> 메인스레드는 "작업결과 나올때까지 기다릴게." 하고 있는 데드락 상태에 빠지게 됩니다.
```swift
var numbers = [0, 1, 2, 3, 4]
let dispatchQueue = DispatchQueue(label: "custom")

print("데드락 발생!")
(0..<numbers.count).forEach { index in
    DispatchQueue.main.sync {
        print(numbers[index])
    }
}
```
![](/assets/img/post/2024/006.png)

## Serial + Async
```swift
var numbers = [0, 1, 2, 3, 4]
let dispatchQueue = DispatchQueue(label: "custom")
print("Serial + Async 실행결과")
(0..<numbers.count).forEach { index in
    dispatchQueue.async {
        print(numbers[index])
    }
}

sleep(5)
```
![](/assets/img/post/2024/007.png)
![](/assets/img/post/2024/008.png)
> serial이니까 한 스레드에 순차적으로 등록하고, 각 작업은 순서 상관없이 리턴됩니다.

## Concurrent + Sync
```swift
var numbers = [0, 1, 2, 3, 4]
let dispatchQueue = DispatchQueue(label: "custom", attributes: .concurrent)

print("Concurrent + Sync 실행결과")
(0..<numbers.count).forEach { index in
    dispatchQueue.sync {
        print(numbers[index])
    }
}
```
![](/assets/img/post/2024/009.png)
![](/assets/img/post/2024/010.png)
> 이런 경우가 있나 싶네요..?

## Concurrent + Async
```swift
var numbers = [0, 1, 2, 3, 4]
let dispatchQueue = DispatchQueue(label: "custom", attributes: .concurrent)

print("Concurrent + Async 실행결과")
(0..<numbers.count).forEach { index in
    dispatchQueue.async {
        print(numbers[index])
    }
}

sleep(5)
```
> numbers의 갯수가 너무 작아서 순차적으로 출력되는 것처럼 보이지만 갯수만 많아지면 순서가 뒤죽박죽이 됩니다.

![](/assets/img/post/2024/011.png)
![](/assets/img/post/2024/012.png)

