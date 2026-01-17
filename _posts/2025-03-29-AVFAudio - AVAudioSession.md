---
title: "[AVFoundation]AVFAudio - AVAudioSession"
date: 2025-03-29 12:00 +0900
categories:
  - 🍎 iOS
  - AVFoundation
tags:
  - AVFoundation
  - AVAudioSession
---


## AVFAudio

![](/assets/img/post/2025/006.png)

**AVFAudio**는 재생, 녹음, 오디오 처리와 같이 앱에서 시스템 오디오 동작을 구성할 때 사용하는 프레임워크 입니다.

아래와 같이 다양한 클래스들이 존재하는데, 오늘은 **AVAudioSeesion**에 대해 학습해보려고 합니다.

![](/assets/img/post/2025/007.png)



![](/assets/img/post/2025/008.png)




## AVAudioSession

![](/assets/img/post/2025/009.png)

![](/assets/img/post/2025/010.png)


> An audio session acts as an intermediary between your app and the operating system — and, in turn, the underlying audio hardware. You use an audio session to communicate to the operating system the general nature of your app’s audio without detailing the specific behavior or required interactions with the audio hardware. You delegate the management of those details to the audio session, which ensures that the operating system can best manage the user’s audio experience.

**AudioSession**은 앱과 운영체제 사이의 **중개자 역할**을 하며, 하드웨어 레벨의 구체적인 동작이나 상호 작용을 설명하지 않고도 쉽게 운영체제와 소통할 수 있도록 합니다. 

그러면 어떤 방식으로 운영체제와 쉽게 소통할 수 있도록 도와줄까요?

이해하기 쉽게 비유하자면, **AVAudioSession** 은 앱에서 **오디오를 어떻게 사용할 지** 환경 설정을 돕는 **오디오 환경 설정 담당자** 입니다.

예를 들어 마이크를 쓸 건지?, 음악을 재생할 건지?, 전화가 오면 오디오를 멈출지 계속할지? 와 같은 정책을 설정할 수 있습니다.

AudioSession이 제공하는 기능은 아래와 같습니다.

- 오디오 활성화 처리
- 오디오 세션 환경 설정
- 오디오 라우팅 처리
- 오디오 인터럽트 처리

### 1. 오디오 활성화 처리 (active / deactive)

앱이 오디오를 사용할 것인지, 사용이 끝났는지에 대한 설정으로 OS에게 이 앱이 오디오를 쓸 것이라고 알리는 것입니다.

코드는 아래와 같이 구성됩니다.

1. `setCategory(_:mode:policy:options:)` 메소드를 통해 앱에서 오디오를 사용할 목적을 설정합니다.
2. `setActive(_:options:)` 메소드를 통해 OS에게 오디오를 활성화하겠다고 알립니다.

> 앱에서 Active를 요청했더라도 만약 **우선순위가 더 높은 오디오 세션이 Active 되어있다면** (ex: incoming call) **Active 요청이 실패합니다.**

```swift
let session = AVAudioSession.sharedInstance()
session.setCategory(.playAndRecord, mode: .default, options: [])
session.setActive(true)
```

3. `setActive(false)`를 통해 OS에게 오디오를 다 사용했다고 알립니다.
> 여러 앱에서 오디오를 동시에 사용하고 있을 경우, 백그라운드에서 재생중이던 앱 사운드가 작아질 수 있는데 내 앱에서 오디오 세션을 비활성화지 않으면 다른 앱들의 사운드가 계속 작아져있는 상태가 되어 사용자 경험에 좋지 않을 수 있습니다.


### 2. 오디오 세션 환경 설정 (category / mode / categoryOptions)

위에서 `setCategory(_:mode:policy:options:)`  메소드를 통해 오디오를 사용할 목적을 설정했었습니다.

첫번째 파라미터로 **Category**를 받고, 두번째 파라미터로 **Mode**, 세번째 파라미터는 옵셔널로 **CategoryOptions** 를 받습니다.

**Category**(카테고리)는 **앱의 오디오 사용 목적**으로 자주 사용하는 Category로는 playback, playAndRecord 가 있습니다.

- **playback**:  **Only 재생**, 무음 모드에서 소리O
- **playAndRecord**: **재생 + 녹음**, 무음 모드에서 소리 O

위 카테고리 2개는 기본값이 **nonmixable**이며 **CategoryOptions** 를 통해 **mixable하게** 만들 수 있습니다.

그 외엔 아래와 같습니다.

- **ambient**: **Only 재생**, 무음 모드에서 소리 X, **mixable**

- **soloAmbient(default)**: **Only 재생**, 무음 모드에서 소리 X, **nonmixable**

- **record**: **Only 녹음**, 재생 불가

- **multiRoute**: **재생 + 녹음**여러 루트로 오디오 출력 시 사용
>  multiRoute 얘는 안써봐서 잘 모르겠음..


| Category      | 무음 모드 or 잠금 상태 | 다른 앱 오디오와 상호작용       | input(recording)/output(playback) |
| ------------- | ---------------------- | ------------------------------- | --------------------------------- |
| ambient       | play X                 | mix                             | output only                       |
| soloAmbient   | play X                 | interrupt                       | output only                       |
| playback      | play O                 | interrupt(옵션을 통해 mix 가능) | output only                       |
| record        | record O               | interrupt                       | input only                        |
| playAndRecord | play O record O        | interrupt(옵션을 통해 mix 가능) | input & output                    |
| multiRoute    | play O                 | interrupt                       | input & output                    |

**Mode**(모드) 는 Category 에 대해 세부적인 모드를 설정해주는 건데, Category와 Mode 간의 호환이 되지 않는 쌍이 존재해 주의해야 합니다.

- videoRecording 은 record, playAndRecord 카테고리가 아니면 에러 발생
- playAndRecord + voicePrompt 에러 발생

자주 사용되는 모드들

- **default**: 특별한 이유가 없다면 이걸로 설정
- **moviePlayback**: 영상 재생 시
- **videoRecording**: 영상 녹화 시
- **spokenAudio**: for continuous spoken audio (ex: 팟캐스트 앱은 이 모드를 사용)
- **voicePrompt**: for TTS (ex: 카플레이)

그 외엔 voiceChat, gameChat, measurement 가 존재합니다.

[**CategoryOptions**](https://developer.apple.com/documentation/avfaudio/avaudiosession/categoryoptions-swift.struct)(옵션)은 다른 앱의 소리와 mix를 할 건지, mix 한다면 밸런스는 어떻게 할 건지 등등 추가적인 옵션을 설정합니다.

- **defaultToSpeaker**: 연결된 오디오 receiver가 따로 있을 때(ex: 이어폰, 카오디오 등) 무시하고 폰 스피커로 오디오를 출력하게 해주는 옵션
- **mixWithOthers**: 다른 앱의 소리과 우리 앱의 소리를 같이 들리게 해주는 옵션
- **duckOthers**: mixWithOthers를 기본으로 하며, 다른 앱의 소리를 줄여줌
- **allowBluetooth**: 블루투스 기기를 input 으로 사용할 수 있도록 하는 옵션
- **allowBluetoothA2DP**: A2DP를 지원하는 블루투스 기기로 재생할 수 있도록 하는 옵션(playback 카테고리에서는 기본 제공이지만, playAndRecord 카테고리에서는 기본이 아니기에 따로 설정해주어야 함.)
- **allowAirPlay**: AirPlay 기기(ex: 애플 TV, 홈팟)로 재생할 수 있도록 하는 옵션
- **interruptSpokenAudioAndMixWithOthers**: mixWithOthers를 기본으로 하며, 다른 앱이 spokenAudio인 경우 내 앱에서만 소리를 재생
- **overrideMutedMicrophoneInterruption**: 마이크가 뮤트되어도 세션이 interrupt되지 않도록 해주는 옵션(ex: 줌 회의 중 마이크를 껐다가 다시 켜도 빠르게 입력될 수 있도록)

### 3.오디오 라우팅 처리

**Audio Route**란 **입력 소스**(내장 마이크, 외부 마이크 등)로부터 **출력 소스**(내장 스피커, 블루투스 이어폰 등)까지의 **경로**를 말합니다.

아이폰 내장 스피커로 음악을 듣다가 에어팟을 연결하는 경우, 바로 이어서 에어팟으로 재생되는 경험을 해봤을텐데 이런 경우가 Audio Route가 변경된 경우입니다.

오디오 input/output이 추가되거나 제거될 때 Audio Route가 변한다고 할 수 있는데, OS(운영체제)로부터 **Notification을 통해 Audio Route가 변경되었음을 감지**할 수 있습니다.

아래는 Audio Route 변경을 감지하는 예시 코드 입니다.

```swift
init() {
    // 노티피케이션 옵저버 등록
    NotificationCenter.default.addObserver(
        self,
        selector: #selector(handleRouteChange(_:)),
         name: AVAudioSession.routeChangeNotification,
         object: nil
    )
}
    
deinit {
    NotificationCenter.default.removeObserver(self)
}

    // 라우팅 변경 처리 함수
    @objc func handleRouteChange(_ notification: Notification) {
        guard let userInfo = notification.userInfo,
              let reasonValue = userInfo[AVAudioSessionRouteChangeReasonKey] as? UInt,
              let reason = AVAudioSession.RouteChangeReason(rawValue: reasonValue) else {
            return
        }

        switch reason {
        case .oldDeviceUnavailable:
            print("이전 오디오 장치가 사용 불가능해졌습니다 (ex. 이어폰 뽑힘)")
            // 일시정지
        case .newDeviceAvailable:
            print("새 오디오 장치가 연결되었습니다 (ex. 블루투스 기기 연결)")
            // 출력 장치 변경
        case .categoryChange:
            print("카테고리 변경")
        default:
            print("기타 라우팅 변경: \(reason.rawValue)")
        }

        // 현재 출력 경로 확인
        let currentRoute = AVAudioSession.sharedInstance().currentRoute
        for output in currentRoute.outputs {
            print("현재 출력 장치: \(output.portType.rawValue)")
        }
    }
}
```

![](/assets/img/post/2025/011.png)

위는 iOS 앱에서 전형적으로 발생하는 Audio Route Change 상황에 대한 플로우입니다.

애플 공식문서에 따르면, 일반적으로 사용자들은 헤드폰을 연결할 때 앱에서 재생하고 있던 콘텐츠를 그대로 이어서 재생하기를 원할 것이고, 반대로 연결을 해제할 때에는 앱에서 재생하고 있던 콘텐츠가 일시정지 되기를 원할 것입니다.(수치사 방지)
> An important behavior related to route changes occurs when a user plugs in or removes a pair of headphones (see Playing audio in [Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines)). When users connect a pair of wired or wireless headphones, they’re implicitly indicating that audio playback should continue, but privately. They expect an app that’s currently playing media to continue playing without pause. However, when users _disconnect_ their headphones, they don’t want to automatically share what they’re listening to with others. Applications should respect this implicit privacy request and automatically pause playback when users disconnect their headphones.

애플에서 제공하는 AVPlayer 를 사용하면 위 동작이 보장되지만, AVPlayer를 사용하지 않는다면 일관된 사용자 경험 제공을 위해 해당 플로우에 대해 개발자가 대응해주어야 합니다.

### 4.오디오 인터럽트 처리

**오디오 인터럽트**(Audio Interrupt)란 우리 앱에서 오디오 세션을 Active하고 재생하는 도중, 우리 앱의 오디오보다 우선순위가 더 높은 오디오 이벤트가 발생할 수 있습니다.(ex: 전화 오는 경우)

이 경우 전화의 우선순위가 더 높기 때문에 앱에서 재생되던 오디오가 일시정지되고 전화벨 소리가 재생됩니다.

이러한 경우를 우리 앱의 입장에서  '**오디오 인터럽트** 가 발생했다.' 라고 합니다.

반대로 다른 앱에서 오디오가 재생 중일 때, 이를 중단시키고 우리 앱의 오디오를 재생시킬 수도 있습니다. 예를 들어 '음악' 앱을 통해 노래를 재생 중일 때, '유튜브 뮤직' 앱을 통해 노래를 재생시키면, '음악' 앱에서 재생 중이던 노래는 일시정지 되고, '유튜브 뮤직' 앱의 노래가 됩니다.
이러한 경우 '유튜브 뮤직' 앱이 '**오디오 인터럽트** 를 발생시켰다.' 라고 합니다.

이처럼 오디오 인터럽트가 발생했을 때와 끝났을 때, 오디오 세션을 통해 **노티피케이션을 감지**하고 그에 맞는 **대응**을 할 수 있습니다.

![](/assets/img/post/2025/012.png)

아래 코드는 오디오 인터럽트에 대해 대응하는 예시 코드입니다.
하지만 인터럽트가 끝났을 때 호출되는 `AVAudioSession.InterruptionType` `.ended` 가 항상 호출된다는 보장이 없기 때문에 주의해야 합니다.

```swift
init() {
        // 인터럽트 노티피케이션 구독
        NotificationCenter.default.addObserver(
            self,
            selector: #selector(handleInterruption(_:)),
            name: AVAudioSession.interruptionNotification,
            object: AVAudioSession.sharedInstance()
        )
    }
    
    deinit {
        NotificationCenter.default.removeObserver(self)
    }
    
    @objc func handleInterruption(_ notification: Notification) {
        guard let userInfo = notification.userInfo,
              let typeValue = userInfo[AVAudioSessionInterruptionTypeKey] as? UInt,
              let type = AVAudioSession.InterruptionType(rawValue: typeValue) else {
            return
        }

        switch type {
        case .began:
            print("🎧 오디오 인터럽트 시작됨 (ex. 전화 수신)")
            // 재생 중단 or 상태 저장
            pauseAudio()

        case .ended:
            print("🎧 오디오 인터럽트 종료됨")

            // 인터럽트 종료 후 재개 가능 여부 확인
            if let optionsValue = userInfo[AVAudioSessionInterruptionOptionKey] as? UInt {
                let options = AVAudioSession.InterruptionOptions(rawValue: optionsValue)
                if options.contains(.shouldResume) {
                    print("👉 오디오 재개 가능, 다시 재생 시작")
                    resumeAudio()
                } else {
                    print("⛔️ 오디오 재개 불가, 사용자가 수동으로 조치해야 함")
                }
            }

        @unknown default:
            break
        }
    }
    
    func pauseAudio() {
        // 예: 오디오 엔진 정지 or 플레이어 일시정지
    }
    
    func resumeAudio() {
        // 예: 오디오 재시작
    }
```

추가로 AVAudioSession은 앱의 요구 사항을 더 디테일하게 반영할 수 있도록 시스템 인터럽트에 대한 동작도 커스터마이징을 제공합니다.

전화 수신과 같은 시스템 알림은 활성화된 오디오 세션을 중단시키는데, 만약 시스템이 앱의 오디오 세션을 중단하지 않도록 하려면 

[`setPrefersNoInterruptionsFromSystemAlerts(_:setPrefersNoInterruptionsFromSystemAlerts(_:) `](https://developer.apple.com/documentation/avfaudio/avaudiosession/setprefersnointerruptionsfromsystemalerts\(_:\)) 메소드를 활용할 수 있습니다.


