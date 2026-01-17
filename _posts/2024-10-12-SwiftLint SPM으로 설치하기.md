---
title: "[Xcode]SwiftLint SPM으로 설치하기"
date: 2024-10-12 12:00 +0900
categories:
  - 🍎 iOS
  - Xcode
tags:
  - SwiftLint
  - SPM
---

신규 프로젝트에서 SwiftLint를 도입하기로 했습니다!
예전에 SPM으로 설치해본 경험이 있지만 너무 오래되서 가물가물하더라구요.
그래서 이번 기회에 정리해놓으려고 합니다.

방식이 두가지가 있습니다.

첫번째는 SwiftLintBuildToolPlugin 을 사용하는 방법입니다.
하지만 이 방식은 build phase에 script를 추가하지 않는 방법이라 커스텀 룰을 적용할 수 없어보여 추천하지 않습니다.

두번째는 brew를 통해 터미널에도 SwiftLint를 설치하고, SPM을 통해 SwiftLint를 설치하는 방법입니다.
이 방식은 build phase에 script를 실행하도록 해 커스텀 룰을 적용할 수 있습니다. 

먼저 간단한 SwiftLintBuildToolPlugin 방식을 알아보겠습니다.


## SwiftLintBuildToolPlugin 방식 (비추천)

### 1. XCode에서 spm으로 SwiftLint 디펜던시 설치
```
https://github.com/realm/SwiftLint
```


![](/assets/img/post/2024/056.png)

![](/assets/img/post/2024/057.png)

이 부분이 중요한데, target을 None으로 설정해주어야 합니다.

![](/assets/img/post/2024/058.png)

![](/assets/img/post/2024/059.png)

아래와 같이 설치되었고, build phase에 가보면 깨끗한 상태입니다.

![](/assets/img/post/2024/060.png)

![](/assets/img/post/2024/061.png)

Run Build Tool Plug-ins phase에 + 버튼을 눌러 SwiftLintBuildToolPlugIn을 추가해줍니다.

![](/assets/img/post/2024/062.png)

그 다음 빌드해보면 정상적으로 SwiftLint가 에러를 잡아주는 걸 볼 수 있습니다.

![](/assets/img/post/2024/063.png)


## SPM + brew 설치 방식 (추천: 커스텀룰 적용 가능)

### 1. XCode에서 spm으로 SwiftLint 디펜던시 설치

spm을 통해 디펜던시를 추가하는 건 위와 같습니다.

### 2. SwiftLint 실행 스크립트 추가

달라지는 부분은 이 부분입니다. 
Run Build Tool Plug-ins phase에 플러그인을 추가해주는 것이 아니라,

![](/assets/img/post/2024/061.png)

+버튼 -> New Run Script Phase를 눌러 새 스크립트 페이즈를 추가합니다.

![](/assets/img/post/2024/065.png)


이름은 원하는대로 설정해도 되는데, SwiftLint를 실행한다는 뜻을 담으면 좋을 것 같습니다.

![](/assets/img/post/2024/066.png)

컴파일 소스 페이즈 뒤에 위치시킨 건 아래 내용을 의식해 옮겨주었습니다.
기존 프로젝트에선 컴파일 소스 페이즈 전에 린트를 실행시켜주었는데,
일단 문서대로 가이드하겠습니다.

![](/assets/img/post/2024/067.png)

> 출처: https://github.com/realm/SwiftLint/blob/main/README_KR.md

그 다음 아래 스크립트를 붙여 넣어줍니다.
대충 swiftlint가 설치되어 있는지 확인하는 분기문같은데, 쉘스크립트 문법은 잘 몰라 넘어가겠습니다.

```shell
if which swiftlint >/dev/null; then
  swiftlint
else
  echo "warning: SwiftLint not installed, download from https://github.com/realm/SwiftLint"
fi
```

![](/assets/img/post/2024/068.png)

Run Script 에는 2개의 옵션이 보이는데, 아래처럼 동작한다는 것 같습니다.

**For install builds only**: Build Action을 취했을때만 스크립트가 실행. Run Action을 취한경우에는 실행되지 않음
**Based on dependency analysis**: 증분빌드시에 변경사항이 없다면 스크립트 실행을 스킵합니다.

### 2.  터미널에 SwiftLint 설치 with homebrew

자신의 맥 터미널에서 brew를 통해 설치해줍니다.

```
brew install swiftlint
```

### 발생할 수 있는 문제 1: homebrew 경로 문제

> 출처: https://github.com/realm/SwiftLint/blob/main/README_KR.md

만약, 애플 실리콘 환경에서 Homebrew를 통해 SwiftLint를 설치했다면, 아마도 다음과 같은 경고를 겪었을 것입니다.

> warning: SwiftLint not installed, download from [https://github.com/realm/SwiftLint](https://github.com/realm/SwiftLint)

그 이유는, 애플 실리콘 기반 맥에서 Homebrew는 기본적으로 바이너리들을 `/opt/homebrew/bin`에 저장하기 때문입니다. SwiftLint가 어디 있는지 찾는 것을 Xcode에 알려주기 위해, build phase에서 `/opt/homebrew/bin`를 `PATH` 환경 변수에 동시에 추가하여야 합니다.

```shell
if [[ "$(uname -m)" == arm64 ]]; then
    export PATH="/opt/homebrew/bin:$PATH"
fi

if which swiftlint > /dev/null; then
  swiftlint
else
  echo "warning: SwiftLint not installed, download from https://github.com/realm/SwiftLint"
fi
```

혹은 아래와 같이 `/usr/local/bin`에 심볼릭 링크를 생성하여 실제 바이너리가 있는 곳으로 포인팅할 수 있습니다.

```shell
ln -s /opt/homebrew/bin/swiftlint /usr/local/bin/swiftlint
```

### 발생할 수 있는 문제 2: 앱 sandbox 문제 

> error: Sandbox: bash(82460) deny(1) file-read-data /Users/~~

![](/assets/img/post/2024/069.png)

User Script Sandboxing을 no 로 설정해주면 해결할 수 있습니다.

> 하지만 보안 측면에서 문제가 발생할 수 있고 App Store 심사 과정에서 반려될 가능성이 있어
> User Script Sandboxing을 no로 하지 않고 해결할 수 있는 방법을 찾는게 좋다고 하네요..!

![](/assets/img/post/2024/070.png)
