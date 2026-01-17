---
title: "[UIKit]View Life Cycle"
date: 2024-08-21 12:00 +0900
categories:
  - 🍎 iOS
  - UIKit
tags:
  - UIKit
  - View Life Cycle
---

## 목차
- ViewController? UIView?
- View Life Cycle 이란?
- 실제 결과 확인

## ViewController? UIView?

iOS 앱은 하나 이상의 뷰로 구성되어 있으며, 뷰는 ViewController 위에 있습니다. 
그리고 ViewController에 UIView 나 UIButton 과 같은 뷰를 올리기 때문에, 대체 뷰 라이프사이클의 뷰는 뭐지? ViewController를 말하는거야? UIView 를 말하는거야? 하고 헷갈리기 쉽습니다.

![](/assets/img/post/2024/025.png)

위 계층 구조에서 보이듯이 뷰는 뷰컨트롤러 위에 존재합니다.
그리고 뷰컨트롤러는 명칭처럼 뷰를 컨트롤하기 위한 객체일 뿐, Life Cycle에서의 뷰는 ViewController의 View를 칭합니다. 🙂

## View Life Cycle 이란?

위에서 언급했듯이 iOS 앱은 하나 이상의 뷰로 구성되어 있으며, 각각의 뷰들은 Life Cycle을 가지고 있습니다. Life Cycle이란 번역하면 **생명 주기** 이고 뷰에 대한 생명주기는 뭘까요?
뷰는 보여졌다 사라졌다 하는데, 보여지고 사라지는 각 단계를 생명주기라고 합니다.

결론부터 보겠습니다.

![](/assets/img/post/2024/026.png)

> https://developer.apple.com/documentation/uikit/uiviewcontroller

뷰컨트롤러의 뷰는 위와 같은 Life Cycle을 가지게 됩니다.

**ViewDidLoad**
- 뷰가 메모리에 올라가는 시점. 
- 화면이 보여질 때 처음 한번만 실행해야 하는 작업들을 주로 여기서 수행합니다.
- 일반적으로 리소스를 초기화하거나 초기화면을 구성할 때 사용합니다.

**ViewWillAppear**
- 뷰가 화면에 나타나기 직전 시점.
- 화면이 나타나는 애니메이션이 나타나기 전에 호출됩니다.
-  화면이 보여지기 직전이므로 애니메이션을 보여주는 작업들을 주로 여기서 수행합니다.

**ViewIsAppearing**
- 뷰가 화면에 나타나기 시작하는 시점?
- 이건 잘 모르겠네요 😂

**ViewDidlAppear**
- 뷰가 화면에 **완전히** 나타난 시점.
-  화면이 나타나는 애니메이션이 완전히 끝난 후에 호출됩니다.
- **따라서, 애니메이션이 중간에 취소되면 DidAppear는 불리지 않습니다.**
- 주로 이곳에서 네트워킹하여 화면을 준비하는 작업들을 수행합니다.

**VIewWillDisAppear**
- 뷰가 사라지기 직전 시점.
- 화면이 사라지는 애니메이션이 나타나기 전에 호출됩니다.

**VIewDidDisAppear**
- 뷰가 화면에서 **완전히** 사라진 시점.
- 화면이 사라지는 애니메이션이 완전히 끝난 후에 호출됩니다.
- **따라서, 사라지는 애니메이션이 중간에 취소되면 DidDisAppear는 불리지 않습니다.**

## 실제 결과 확인

![](/assets/img/post/2024/027.gif)

```
// 앱이 시작되고 A뷰가 화면에 나타남
A: viewDidLoad

A: viewWillAppear
A: viewIsAppearing
A: viewDidAppear

// 탭바를 눌러 B뷰가 화면에 나타남
B: viewDidLoad

B: viewWillAppear
A: viewWillDisappear
B: viewIsAppearing
A: viewDidDisappear
B: viewDidAppear

// 탭바를 눌러 A뷰가 화면에 나타남
A: viewWillAppear
B: viewWillDisappear
A: viewIsAppearing
B: viewDidDisappear
A: viewDidAppear

```

## A에서 B가 나타날때

| A                 | B                             |
| ----------------- | ----------------------------- |
|                   | viewDidLoad(처음 보여질 때만)     | 
|                   | viewWillAppear                |
| viewWillDisAppear |                               |
|                   | viewIsAppearing               |
| viewDidDisappear  |                               |
|                   | viewDidAppear                 |
|                   |                               |

B가 나타날 준비와 A가 사라질 준비를 하고, B가 나타나는 동안 A가 사라지고 B가 나타나네요.

## B에서 A가 나타날 때

| A                                             | B                 |    
| --------------------------------------------- | ----------------- | 
| viewDidLoad 호출 X (이미 메모리에 올라가있음)         |                   |     
| viewWillAppear                                |                   |    
|                                               | viewWillDisAppear |     
| viewIsAppearing                               |                   |     
|                                               | viewDidDisappear  |     
| viewDidAppear                                 |                   |    
|                                               |                   |     

반대로 해보면 A가 나타날 준비와 B가 사라질 준비를 하고, A가 나타나는 동안 B가 사라지고 A가 나타납니다. 둘이 일치하네요!


코드는 아래와 같습니다.

```swift

import UIKit

class AViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        print("A: viewDidLoad")
    }
    override func viewWillAppear(_ animated: Bool) {
        print("A: viewWillAppear")
    }
    override func viewIsAppearing(_ animated: Bool) {
        print("A: viewIsAppearing")
    }
    override func viewDidAppear(_ animated: Bool) {
        print("A: viewDidAppear")
    }
    override func viewWillDisappear(_ animated: Bool) {
        print("A: viewWillDisappear")
    }
    override func viewDidDisappear(_ animated: Bool) {
        print("A: viewDidDisappear")
    }
}

class BViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        print("B: viewDidLoad")
    }
    override func viewWillAppear(_ animated: Bool) {
        print("B: viewWillAppear")
    }
    override func viewIsAppearing(_ animated: Bool) {
        print("B: viewIsAppearing")
    }
    override func viewDidAppear(_ animated: Bool) {
        print("B: viewDidAppear")
    }
    override func viewWillDisappear(_ animated: Bool) {
        print("B: viewWillDisappear")
    }
    override func viewDidDisappear(_ animated: Bool) {
        print("B: viewDidDisappear")
    }
}

```

