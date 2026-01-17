---
title: "[Swift]Swift Concurrency - Sendable"
date: 2024-10-25 12:00 +0900
categories:
  - 🍎 iOS
  - Swift
tags:
  - Swift
  - Concurrency
  - Sendable
---

Swift6부터 Sendable 관련 경고나 에러가 너무 많이 뜨더라구요.
Concurrency도 모르고 Sendable도 모르는데..!
그래서 Sendable 애플 공식문서와 swift docs를 읽고 감을 익혀보려고 합니다.
먼저 애플 공식문서부터 읽어볼게요!

애플 공식문서 링크: https://developer.apple.com/documentation/swift/sendable
Swift docs 링크: https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/#Sendable-Types

## Sendable 이란?

```swift
protocol Sendable
```

데이터 레이스 위험(a risk of data races) 없이 임의의 동시 컨텍스트(arbitrary concurrent contexts)에서 값을 공유할 수 있는 스레드 세이프(thread-safe)한 형식입니다.

> A thread-safe type whose values can be shared across arbitrary concurrent contexts without introducing a risk of data races.

## Overview 개요

타입의 값(value of the type)은 공유 변경 가능한 상태(mutable state)를 갖지 않을 수도 있고, 잠금으로 또는 특정 액터에서만 접근하도록 강제하여 해당 상태를 보호할 수도 있습니다.

> Values of the type may have no shared mutable state, or they may protect that state with a lock or by forcing it to only be accessed from a specific actor.

한 동시성 도메인(one concurrency domain)에서 다른 동시성 도메인으로 sendable 타입의 값을 안전하게 전달할 수 있습니다. 예를들어, 액터의 메소드를 호출할 때 sendable 값을 인자(argument)로 전달할 수 있습니다.

> You can safely pass values of a sendable type from one concurrency domain to another — for example, you can pass a sendable value as the argument when calling an actor’s methods.

> 🤔 전 sendable 이라는 네이밍을 이해할 때, 안전하게 전달가능하니까 전달가능하단 의미로 sendable이라고 이해했어요. 이렇게 생각하면 조금은 이해가 쉬운 느낌?

아래 목록들은 sendable 타입을 채택(can be marked as sendable)할 수 있습니다.

- 값 타입 (Value Types)
- 변경 가능한 저장소가 없는 참조 형식 (Reference types with no mutable storage)
- 상태에 대한 접근을 내부적으로 관리하는 참조 형식 (Reference types that internally manage access to their state)
- 함수와 클로저(`@Sendable`로 표시하여) (Functions and closures (by marking them with @Sendable)

이 프로토콜에는 필요한 메소드나 속성이 없지만, 컴파일 타임에 강제(enforced)되는 의미적 요구사항(semantic requirements)이 있습니다. 이러한 요구사항은 아래 섹션에 나열되어 있습니다. Sendable에 대한 규칙은 타입의 선언과 동일한 파일에서 선언되어야 합니다.

> Although this protocol doesn’t have any required methods or properties, it does have semantic requirements that are enforced at compile time. These requirements are listed in the sections below. Conformance to Sendable must be declared in the same file as the type’s declaration.

> 🤔 semantic requirements를 어떻게 번역해야할지 모르겠는데, 컴파일 타임에 체킹?한다는 내용인 것 같아요. Swift5 에서 6로 변경하면 컴파일 타임에 경고와 에러가 엄청 발생하는데 그 내용인 듯 합니다.

컴파일러 개입(경고 or 에러)(의역: compiler enforcement) 없이 Sendable 채택을 선언하려면, `@unchecked Sendable` 을 사용합니다. 예를 들어, lock 또는 큐로 해당 상태에 대해 모든 접근을 보호하여 확인되지 않은 sendable(unchecked sendable) 타입의 정확성에 대한 책임을 가집니다.
unchecked Sendable 사용은 같은 파일 내에 있어야 한다는 룰도 비활성화 시킵니다. 

> To declare conformance to Sendable without any compiler enforcement, write @unchecked Sendable. You are responsible for the correctness of unchecked sendable types, for example, by protecting all access to its state with a lock or a queue. Unchecked conformance to Sendable also disables enforcement of the rule that conformance must be in the same file.

> 🤔 컴파일러 경고를 안 받고 싶으면, @unchecked Sendable을 붙이고 대신 책임을 너가 지라는 뜻인 것 같아요.

`Task`가 속한 언어 수준 동시성 모델에 대한 자세한 내용은 [Swift 프로그래밍 언어](https://docs.swift.org/swift-book/)의 [동시성](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/)을 참조하세요.

> For information about the language-level concurrency model that Task is part of, see Concurrency in The Swift Programming Language

## Sendable Structures and Enumerations 구조체 및 열거형

Sendable 프로토콜의 요구사항을 만족하려면, enum 또는 struct의 모든 멤버 변수(members)와 연관 값(associated values)들이 sendable을 채택해야 합니다. 어떤 경우에는 요구사항을 만족하는 struct와 enum이 암시적(implicitly)으로 다음을 준수합니다.

> To satisfy the requirements of the Sendable protocol, an enumeration or structure must have only sendable members and associated values. In some cases, structures and enumerations that satisfy the requirements implicitly conform to Sendable

- Frozen Struct 와 Enum
- public이 아니며 `@usableFromInline`로 사용되지 않는 struct 및 enum

그렇지 않으면, sendable을 명시적으로 준수해야 합니다.

non-sendable 한 저장 프로퍼티를 가지는 struct 와 non-sendable 한연관 값을 가진 enum은  `@unchecked Sendable` 을 사용할 수 있습니다. sendable의 의미론적 요구사항을 만족하는지 수동으로 검증한 후, 컴파일 타임에 정확성 검사를 하지 않도록 비활성화 할 수 있습니다.

> Otherwise, you need to declare conformance to Sendable explicitly.
> Structures that have nonsendable stored properties and enumerations that have nonsendable associated values can be marked as @unchecked Sendable, disabling compile-time correctness checks, after you manually verify that they satisfy the Sendable protocol’s semantic requirements.

> 🤔 구조체와 열거형 모두 명시적이거나 암시적으로 sendable 한 멤버들을 가져야 하는데, 그렇지 않을 경우 수동으로 확인하고 @unchecked Sendable을 붙여서 경고 안나게 해라~ 는 뜻인것 같네요!

## Sendable Actors 액터

모든 Actor 타입은 암시적으로 Sendable을 준수합니다. 변경 가능한 상태에 대한 모든 접근이 순차적으로 수행되도록 보장하기 때문입니다.

> All actor types implicitly conform to Sendable because actors ensure that all access to their mutable state is performed sequentially.

## Sendable Classes 클래스

Sendable 을 준수하려면 Class 는 다음을 수행해야 합니다.

> To satisfy the requirements of the Sendable protocol, a class must:

- `final`으로 표시되어야 합니다. (Be marked final)
- immutable하고 sendable한 저장 프로퍼티만 포함해야 합니다. (Contain only stored properties that are immutable and sendable)
- 슈퍼클래스가 없거나 슈퍼클래스로 `NSObject` 가 있어야 합니다. (Have no superclass or have NSObject as the superclass)

`@MainActor` 로 표시된 class는 암시적으로 sendable 합니다. 메인 액터가 상태에 대한 모든 접근을 조정하기 때문입니다. 이러한 클래스에는 변경 가능하고 전송할 수 없는 저장된 속성이 있을 수 있습니다.

> Classes marked with @MainActor are implicitly sendable, because the main actor coordinates all access to its state. These classes can have stored properties that are mutable and nonsendable.

위의 요구사항을 충족하지 않는 class는 sendable 프로토콜의 의미론적 요구사항을 만족하는지 수동으로 검증한 후, 컴파일 타임에 정확성 검사를 하지 않도록 비활성화 할 수 있습니다.

> Classes that don’t meet the requirements above can be marked as @unchecked Sendable, disabling compile-time correctness checks, after you manually verify that they satisfy the Sendable protocol’s semantic requirements.

> 🤔 아직 메인 액터가 감이 잘 안오네요. 클래스인데 immutable 하고 sendable한 저장 프로퍼티만 포함해야한다니 마이그레이션이 쉽지 않을 것 같습니다..ㅋㅋㅋ

## Sendable Functions and Closures 함수 및 클로저

sendable 프로토콜을 따르는 대신에 `@Sendable` 어트리뷰트로 sendable 함수와 클로저를 명시할 수 있습니다. 함수 또는 클로저가 캡쳐하는 모든 값은 sendable 해야 합니다. 또한 sendable clousures는 by-value 캡쳐만 사용해야 하며 캡쳐된 값은 sendable 해야 합니다.

> Instead of conforming to the Sendable protocol, you mark sendable functions and closures with the @Sendable attribute. Any values that the function or closure captures must be sendable. In addition, sendable closures must use only by-value captures, and the captured values must be of a sendable type.

sendable 클로저를 예상하는 컨텍스트에서 요구사항을 만족하는 클로저는 암시적으로 sendable을 준수합니다. (예시, Task.detached(priority:operation:) 호출)

> In a context that expects a sendable closure, a closure that satisfies the requirements implicitly conforms to Sendable — for example, in a call to Task.detached(priority:operation:)

타입 어노테이션의 일부로 `@Sendable` 을 사용하거나 클로저의 매개변수 앞에 `@Sendable` 을 사용하여 해당 클로저를 명시적으로 sendable 하도록 할 수 있습니다. - 예를 들면 다음과 같습니다.

> You can explicitly mark a closure as sendable by writing @Sendable as part of a type annotation, or by writing @Sendable before the closure’s parameters — for example:

```swift
let sendableClosure = { @Sendable (number: Int) -> String in
    if number > 12 {
        return "More than a dozen."
    } else {
        return "Less than a dozen"
    }
}
```

## Sendable Tuples 튜플

Sendable 프로토콜의 요구사항을 만족하려면 튜플의 모든 요소가 sendable 해야 합니다. 요구사항을 만족하는 튜플은 암시적으로 Sendable 을 준수합니다.

> To satisfy the requirements of the Sendable protocol, all of the elements of the tuple must be sendable. Tuples that satisfy the requirements implicitly conform to Sendable

## Sendable Metatypes 메타타입

Int.Type과 같은 메타타입은 암시적으로 Sendable 프로토콜을 준수합니다.
> Metatypes such as `Int.Type` implicitly conform to the `Sendable` protocol.


여기까지가 [sendable 애플 공식문서](https://developer.apple.com/documentation/swift/sendable)의 내용이었습니다.

--- 

여기서부턴 [sendable swift 공식문서](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/#Sendable-Types)의 내용입니다.

애플 공식문서에서 언급된 내용들을 좀 더 상세하게 설명해주고 있는데, 겹치는 내용이 많아 따로 보면 좋을만한 부분만 추려봤습니다. 전체 내용은 [sendable swift 공식문서](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/#Sendable-Types) 여기서 확인할 수 있습니다.

## 동시성 도메인

task 또는 actor의 인스턴스 내부에서 변수나 프로퍼티같은 변경가능한 상태를 포함하는 프로그램의 일부를 동시성 도메인이라고 합니다.

> Inside of a task or an instance of an actor, the part of a program that contains mutable state, like variables and properties, is called a concurrency domain.

## 예시

sendable한 프로퍼티만 가지는 struct나 sendable한 연관 값만 가지는 enum 처럼, 일부 타입은 항상 sendable 합니다. 예를 들어

> Some types are always sendable, like structures that have only sendable properties and enumerations that have only sendable associated values. For example:

```swift
struct TemperatureReading: Sendable {
    var measurement: Int
}


extension TemperatureLogger {
    func addReading(from reading: TemperatureReading) {
        measurements.append(reading.measurement)
    }
}


let logger = TemperatureLogger(label: "Tea kettle", measurement: 85)
let reading = TemperatureReading(measurement: 45)
await logger.addReading(from: reading)
```

TemperatureReading 는 sendable한 프로퍼티만 가지는 struct이고, public 또는 @usableFromInline으로 명시되어(makred) 있지 않기 때문에, 암시적으로 sendable 합니다.

> Because TemperatureReading is a structure that has only sendable properties, and the structure isn’t marked public or @usableFromInline, it’s implicitly sendable. 

아래 코드는 Sendable 프로토콜을 암시적으로 준수하는 struct의 버전입니다.

> Here’s a version of the structure where conformance to the Sendable protocol is implied:

```swift
struct TemperatureReading {
    var measurement: Int
}
```

타입을 sendable로 명시적으로 표시하고, Sendable 프로토콜에 대한 암시적인 준수를 오버라이딩(overriding) 하려면 extension을 사용하면 됩니다.

> To explicitly mark a type as not being sendable, overriding an implicit conformance to the Sendable protocol, use an extension:

```swift
struct FileDescriptor {
    let rawValue: CInt
}

@available(*, unavailable)
extension FileDescriptor: Sendable { }
```

위 코드는 POSIX file descriptor를 감싸는 래퍼의 일부를 보여줍니다. 파일 디스크립터에 대한 인터페이스는 sendable한 integer를 사용하여 열린 파일(open files)을 식별(identify)하고 다룰 수 있도록(interact)합니다. 하지만 파일 디스크립터는 동시성 도메인을 통해 보내는 것이 안전하지 않습니다.  

> The code above shows part of a wrapper around POSIX file descriptors. Even though interface for file descriptors uses integers to identify and interact with open files, and integer values are sendable, a file descriptor isn’t safe to send across concurrency domains.

위의 코드에서 FileDecriptor는 암시적으로 sendable 기준을 충족하는 struct입니다. 그러나 위 extension은 Sendable을 사용할 수 없도록 하여 타입이 Sendable 해지는 것을 방지합니다.

> In the code above, the FileDescriptor is a structure that meets the criteria to be implicitly sendable. However, the extension makes its conformance to Sendable unavailable, preventing the type from being sendable.

> 🤔 이 코드가 잘 이해가 가지않았는데, ChatGPT와 함께 해석해보니 조금은 이해가 가는 것 같습니다. 위 코드는 파일디스크립터에 대한 작은 코드조각일 뿐이고, 파일디스크립터의 역할이 무엇인지 생각해보면 Sendable하지 않도록 해야하는 이유가 되는 것 같습니다. 아래는 ChatGPT의 설명입니다.

### ChatGPT 의 설명
이 코드는 POSIX 파일 디스크립터를 감싸는 구조체 `FileDescriptor`의 일부입니다. 파일 디스크립터는 운영체제가 열려 있는 파일을 식별하고 다룰 수 있게 해 주는 "정수"입니다. 이 정수(`CInt`) 값 자체는 "전송 가능(Sendable)"한 타입이지만, 이 코드에서는 `FileDescriptor`가 실제로는 안전하게 전송될 수 없도록 `Sendable`을 막고 있습니다.

#### 핵심 개념 설명

1. **Sendable의 의미**:
    
    - Swift에서 `Sendable` 프로토콜은 데이터가 안전하게 여러 스레드나 비동기 작업 간에 이동될 수 있음을 보장합니다.
    - 기본 정수 타입(`Int`, `CInt`)은 Sendable을 만족하므로, 숫자 값만을 주고받는 경우 스레드 간 전송이 안전합니다.
2. **왜 `FileDescriptor`를 Sendable에서 막는가**:
    
    - 비록 `rawValue`(정수 값) 자체는 Sendable이지만, 파일 디스크립터는 단순한 값 이상입니다. 파일 디스크립터는 실제 파일에 대한 "연결"을 나타내므로, 하나의 스레드나 작업이 파일을 닫거나 변경하면 다른 스레드에서 더 이상 해당 디스크립터를 사용할 수 없습니다.
    - 이를 방지하기 위해 `FileDescriptor`의 `Sendable` 준수를 명시적으로 막아 다른 스레드에 안전하게 전달되지 않도록 한 것입니다.
3. **`@available(*, unavailable)`의 의미**:
    
    - `@available(*, unavailable)`는 코드에서 특정 기능이나 프로토콜 준수를 사용할 수 없도록 만듭니다.
    - 이 예제에서는 `FileDescriptor`의 `Sendable` 준수를 사용 불가능하게 해 놓았으므로, 실수로 `FileDescriptor`를 여러 스레드에서 전송하려 할 때 컴파일러가 경고를 줄 수 있습니다.

결국 이 코드는 파일 디스크립터를 안전하게 다루기 위한 조치입니다. `FileDescriptor`는 비록 "정수"라는 Sendable한 값으로 구성되어 있지만, 여러 스레드 간 안전하게 공유되거나 전송할 수 있는 대상이 아니기 때문에 Sendable을 사용하지 못하게 한 것입니다.