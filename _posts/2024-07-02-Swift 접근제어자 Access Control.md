---
title: "[Swift]Swift 접근제어자 Access Control"
date: 2024-07-02 12:00 +0900
categories:
  - 🍎 iOS
  - Swift
tags:
  - Swift
  - Access Control
---

우선 우리가 잘 알고 있듯이 swift의 접근제어자는 다음과 같이 6개가 있습니다. 5개라고 알고 있는 사람도 많겠지만, Swift 5.9 부터 package 접근제어자가 추가되어 6개가 되었습니다.

> Swift는 코드 내의 엔티티에 대해 6가지 다른 **액세스 수준**을 제공합니다. 이러한 액세스 수준은 엔티티가 정의된 소스 파일, 소스 파일이 속한 모듈 및 모듈이 속한 패키지와 관련이 있습니다.

## 접근 제어자
1. **open**: 가장 높은 수준의 접근 제어자. 다른 모듈에서도 접근, 서브클래싱, 오버라이딩 가능

```swift
// 모듈: ModuleA
open class Animal {
	open fucn bark() {
		print("으르렁")
	}
}
// 모듈: ModuleB
class Cat: Animal {
	override open func bark() {
		print("야옹")
	}
}
```

2. **public**: open과 같은 수준의 접근 제어자. 다른 모듈에서도 접근이 가능하나, 서브클래싱과 오버라이딩은 불가능

```swift
// 모듈: ModuleA
public class Animal {
	public fucn bark() {
		print("으르렁")
	}
}
// 모듈: ModuleB
class Cat: Animal { // Error: 외부모듈에서 서브클래싱 X
	override open func bark() { // Error: 외부모듈에서 오버라이딩 X
		print("야옹")
	}
}
```

3. **internal**: 같은 모듈 내에서는 어디서든지 접근 가능하지만, 외부 모듈에선 접근 불가능

```swift
// 모듈: ModuleA
internal class Animal {
	internal fucn bark() {
		print("으르렁")
	}
}
// 모듈: ModuleB
let animal = Animal() // Error: 외부 모듈에선 접근 불가능
animal.bark()
```

4. **fileprivate**: private 수준이지만 같은 파일 내에서는 접근이 가능
5. **private**: 가장 낮은 수준의 접근 제어자. 해당 요소가 선언된 블록 내에서만 사용할 수 있음.

```swift
// 모듈: ModuleA, 파일: Animal.swift
class Dog {
    private func bark() {
        print("멍멍")
    }
}
class Cat {
    fileprivate func bark() {
        print("야옹")
    }
}

// 모듈: ModuleA, 파일: Zoo.swift
class Zoo {
    let dog = Dog()
    let cat = Cat()
    
    func sound() {
        dog.bark() // Error: private 은 정의된 블록에서만
        cat.bark() // Error: fileprivate은 같은 파일에서만
    }
}

```


## 6. package 접근 제어자 (New)

**package**: 같은 패키지 내에선 접근 가능

새로 생긴 접근제어자라 예시를 들어 설명해보겠습니다.
플레이어를 재생시키는 건 플레이어화면에서만 가능합니다.
![pack](https://cdn.discordapp.com/attachments/1015566062034628609/1257621632109903932/image.png?ex=668512ea&is=6683c16a&hm=8c682def010c62f675cd4a0b7c5cff5559b586252e7385fa3c8903e27e64881e&)
package 접근제어자가 없다면 다른 모듈에서 사용하기 위해 public 접근 제어자를 썼을겁니다. 하지만 public 은 모든 모듈에서 접근 가능하기 때문에 원하지 않는 모듈에서 접근해 사이드 이펙트를 일으킬 수 있습니다.

**구조를 보니 같은 패키지끼리는 접근가능하지만 다른 패키지에선 접근하지 못하게 하면 되지 않을까?**
그래서 나온게 package 접근제어자 입니다.
![pack2](https://cdn.discordapp.com/attachments/1015566062034628609/1257622512234139738/image.png?ex=668513bc&is=6683c23c&hm=61a28ba2400984f620b48ca8698604f3846b227fa8ab8b2de829fc01373df73f&)
모듈을 빌드할때 새로운 플래그 **-package-name**가 전달됩니다.
```swift
swiftc -module-name App -package-name appPkg ... 
swiftc -module-name Player -package-name featurePkg ... 
swiftc -module-name Present -package-name featurePkg ...
```
모듈 빌드가 완료되면, App모듈은 appPkg로 기록되고, player present 모듈은  featurePkg로 기록됩니다.
따라서 패키지 명이 다르기 때문에 App 모듈에서 package 접근제어자에 접근이 불가능해집니다.