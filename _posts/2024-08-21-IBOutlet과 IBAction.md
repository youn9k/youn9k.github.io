---
title: "[UIKit]IBOutlet과 IBAction"
date: 2024-08-21 12:00 +0900
categories:
  - 🍎 iOS
  - UIKit
tags:
  - UIKit
  - IBOutlet
  - IBAction
---

UIKit을 처음 접할 때, 스토리보드는 구닥다리다, 협업에 불편하다 등 악평을 많이 들어 코드베이스로 입문했었습니다. 그리고 이번에 스토리보드를 사용해보게 되었고, 이번 기회에 스토리보드와 코드간의 연결통로? 역할을 해주는 IBAction 과 IBOutlet이 무엇인지 학습해보기로 했습니다.

## 목차
- IB(Interface Builder)
- IBOutlet
- IBAction

## IB(Interface Builder)

IBOutlet, IBAction 둘다 앞에 IB 라는 접두사가 붙는데 이게 뭔지 궁금했습니다.
IB 는 Interface Builder (인터페이스 빌더) 의 줄임말이고,
인터페이스 빌더는

> Xcode에서 사용자 인터페이스(UI, User Interface)를 만들기 위한 그래픽 환경.
> The graphical environment for building a user interface (UI) in Xcode.
> 출처: https://yagom.net/docs/interface-builder/

라고 합니다.
쉽게 말해 코드 작성 대신 GUI에서 마우스 딸깍딸깍으로 UI를 만들 수 있게 도와주는 툴이라고 볼 수 있겠네요! 

![](/assets/img/post/2024/013.png)

## IBOutlet

스토리보드로 UI를 작성하고, 필요한 로직들을 코드로 작성하다보면 이러한 질문들을 마주하게 됩니다.

1. 이 코드가 스토리보드의 어떤 객체를 가리키는가?
2. 스토리보드의 View(UILabel, UIImageView)에 접근하려면 어떤 변수로 접근해야 하는가?

이러한 질문들을 해소해주는 것이 IBOutlet 입니다.
IBOutlet은 코드와 스토리보드의 View객체(UILabel, UIImageView 등)를 연결해줍니다.

대충 아래와 같은 `main.storyboard` 스토리보드가 있다고 할때,

![](/assets/img/post/2024/014.png)

Open As -> Source Code 를 눌러보면..

![](/assets/img/post/2024/015.png)

xml 형태로 볼수 있게 됩니다..
애초에 스토리보드는 xml로 작성되어 있고, XCode 상에서 GUI로 보여줄 뿐입니다.

![](/assets/img/post/2024/016.png)

이 코드가 어떻게 스토리보드의 객체를 가리킬 수 있을까요?

![](/assets/img/post/2024/017.png)

스토리보드에서 outlet을 검색해보면 connections 안에 연결 정보가 저장되어 있음을 알 수 있습니다.

![](/assets/img/post/2024/018.png)

outlet의 destination은 뭘 가르키는걸까? 확인해보겠습니다.

![](/assets/img/post/2024/019.png)

showButton과 연결된 스토리보드의 객체를 가리키고 있네요!
id 는 해당 outlet 연결정보의 고유 id 였습니다.

![](/assets/img/post/2024/020.png)

## IBAction

IBAction은 스토리보드의 View 객체가 이벤트를 발생시켰을 때 동작할 코드를 연결합니다.

IBAction도 IBOutlet과 비슷하게 커넥션에 들어있지 않을까요? 확인해보겠습니다.

맞았네요!

![](/assets/img/post/2024/021.png)

스토리보드의 버튼 객체 안에 connections 안에 action 이란 이름으로 연결 정보가 저장되어 있습니다. 

![](/assets/img/post/2024/022.png)

selector 는 내가 지정한 IBAction의 메소드명이고, 

![](/assets/img/post/2024/023.png)

destination 은 해당 메소드가 있는 뷰컨트롤러의 id였습니다.

![](/assets/img/post/2024/024.png)

종합해보면 스토리보드의 버튼 객체의 특정 이벤트 (사진에선 TouchUpInside)가 발생하면, destination(뷰컨트롤러 id)의 selector(실행할 메소드명)를 찾아가는 구조로 보입니다. 👀✨

IBOutlet 과 IBAction 모두 스토리보드에 **고유한 id를 가진 연결정보**로서 기록되어 있고,
destination, selector 등을 통해 실행할 메소드나 (연결된 변수 - 스토리보드 객체)를 찾아갈 수 있도록 해주고 있었네요!