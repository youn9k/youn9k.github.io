---
title: "[WWDC]WWDC25 - FoundationModels로 엿본 애플이 꿈꾸는 개인화 AI의 미래"
date: 2025-06-24 12:00 +0900
categories:
  - 🍎 iOS
  - Swift
tags:
  - WWDC
  - AI
---

올해 WWDC25에서 가장 궁금하던 부분은 "애플은 AI에 어떻게 대응할까?" 였습니다.
ChatGPT나 Gemini처럼 클라우드 기반 LLM이 각광받는 시대에 개인정보 보호와 기기간 통합을 중시하는 애플이 어떻게 대응할지 궁금했는데요. 바로 이번에 발표된 FoundationModels을 통해 애플의 스탠스를 확인할 수 있었습니다.


이미 과열된 LLM 경쟁에 후발주자로 뛰어들기보다는, 자신들이 오랜 시간 축적해온 사용자 데이터와 프라이버시 보호 철학을 바탕으로 온디바이스 기반의 개인화된 AI를 만들겠다는 애플은 최근 Apple Inteligence의 출시를 미루고 있습니다.

아래는 애플 부사장과의 인터뷰를 담은 영상으로, Apple Inteligence의 출시가 미뤄지고 있는 이유에 대해 설명합니다.
https://youtu.be/pfufC8mcoUo?si=qs3qW8tUxnl_oFWs
> 출처: 유튜브 '오목교 전자상가'

이대로라면 골든 타임을 놓칠 수 있겠다고 생각한 것인지, Apple Inteligence 출시 전에 개발자들로 항하여금 먼저 온디바이스 AI 모델을 사용하여 개발할 수 있도록 'FoundationModels' 이란 프레임워크를 공개했습니다.

이 프레임워크는 iOS, iPadOS, macOS, visionOS를 포함하는 Apple 전 플랫폼에서 온디바이스 언어 모델을 활용할 수 있도록합니다. 모델은 기기에서 오프라인으로 작동하고 운영체제에 내장되어 있어 앱 용량을 늘리지 않으며, 사용자 데이터의 개인정보 보호를 보장합니다.

## FoundationModels의 주요 구성과 코드 활용 예시

FoundationModels는 단순히 온디바이스 모델을 호출하는 수준을 넘어, **프롬프트 엔지니어링부터 구조화된 응답, 도구 호출, 스트리밍 출력** 그리고 성능 최적화관련 API 까지 제공합니다.

![](/assets/img/post/2025/080.png)

프롬프트를 바꾸고 결과를 확인하기 위해 매번 빌드할 필요가 없습니다. FoundationModels는 Playground 환경에서도 동작하기에 프롬프트 엔지니어링에 최적화된 환경을 구성할 수 있습니다. 

```swift
import FoundationModels

let session = LaguageModelSession()
let response = try await session.respond(to: "바르셀로나에서 3일간 여행 일정을 추천해줘")

print(response)
```

### 구조화된 데이터가 필요하다면?

@Generable과 @Guide 매크로를 통해 응답의 형식을 제어할 수 있습니다.
Generable의 유일한 요구사항은 속성도 Generable해야한다는 것입니다.
Guide 매크로는 속성 값에 대해 제약조건을 추가하는 것으로, 예를 들어 설명이란 속성이 있다면 타입이 String이기에 어떤 문자열도 들어올 수 있지만, Guide를 통해 항상 '~입니다.' 로 끝나도록 제약조건을 추가할 수 있습니다.

이를 통해 앱에서 사용하기 편한 형태로 응답을 얻을 수 있습니다.

```swift
@Generable
struct Itinerary: Equatable {
    @Guide(description: "An exciting name for the trip.")
    let title: String
    @Guide(.anyOf(ModelData.landmarkNames))
    let destinationName: String
    let description: String
    @Guide(description: "An explanation of how the itinerary meets the user's special requests.")
    let rationale: String
    
    @Guide(description: "A list of day-by-day plans.")
    @Guide(.count(3))
    let days: [DayPlan]
}

@Generable
struct DayPlan: Equatable {
    @Guide(description: "A unique and exciting title for this day plan.")
    let title: String
    let subtitle: String
    let destination: String

    @Guide(.count(3))
    let activities: [Activity]
}
```

### 외부 도구가 필요하다면?

![](/assets/img/post/2025/081.png)

예를 들어 사용자의 일정 정보를 검색해야한다거나 연락처에 접근해야 하는 등 모델이 외부 소스에 접근해야하는 경우가 있습니다.
이 때 개발자는 'Tool' 프로토콜을 채택하는 사용자 지정 도구(Tool)를 생성할 수 있습니다.
고유한 도구 이름, 도구를 호출할 시기에 대한 자연어 설명, 그리고 모델이 도구를 호출하는 방식을 정의하는 `call()` 함수가 포함됩니다. 

예를 들어, MapKit과 같은 서비스와 연동하여 특정 랜드마크에 대한 관심 지점을 가져오는 도구를 만들 수 있습니다. 

중요한 점은 도구를 호출할 시기와 방법 등은 모델이 자체적으로 결정하며, 개발자가 명시적으로 지정할 수 없다고 합니다.

```swift
import FoundationModels
import SwiftUI

@Observable
final class FindPointsOfInterestTool: Tool {
    let name = "findPointsOfInterest"
    let description = "Finds points of interest for a landmark."
    
    let landmark: Landmark
    
    @MainActor var lookupHistory: [Lookup] = []
    
    init(landmark: Landmark) {
        self.landmark = landmark
    }

    @Generable
    enum Category: String, CaseIterable {
        case hotel
        case cafe
        case museum
        ...
    }

    @Generable
    struct Arguments {
        @Guide(description: "This is the type of destination to look up for.")
        let pointOfInterest: Category

        @Guide(description: "The natural language query of what to search for.")
        let naturalLanguageQuery: String
    }
    
    @MainActor func recordLookup(arguments: Arguments) {
        lookupHistory.append(Lookup(history: arguments))
    }
    
    func call(arguments: Arguments) async throws -> ToolOutput {
        // This sample app pulls some static data. Real-world apps can get creative.
        await recordLookup(arguments: arguments)
        let results = mapItems(arguments: arguments)
        return ToolOutput(
            "There are these \(arguments.pointOfInterest) in \(landmark.name): \(results.joined(separator: ", "))"
        )
    }
    
    private func mapItems(arguments: Arguments) -> [String] {
        suggestions(category: arguments.pointOfInterest)
    }
}
```

### 모델의 사용 가능 여부를 처리하는 방법

![](/assets/img/post/2025/082.png)

온디바이스 AI이기 때문에 모델이 항상 준비된 것은 아닙니다. 예를 들어 기기가 지원하지 않거나, Apple Inteligence가 비활성화되어 있다거나 모델이 아직 다운로드되지 않은 경우 에러 처리를 해주어야 합니다.

```swift
import FoundationModels
import SwiftUI

let model = SystemLanguageModel.default

switch model.availability {
case .unavailableDevice:
  Text("이 기능은 현재 기기에서 지원되지 않습니다.")
case .notEnabled:
  Text("Apple Intelligence를 활성화해야 이 기능을 사용할 수 있습니다.")
case .notReady:
  ProgressView("모델을 준비 중입니다. 잠시만 기다려주세요.")
case .available:
  MyAIFeatureView()
}
```

### 스트리밍 출력으로 사용자 경험 개선하기

![](/assets/img/post/2025/083.gif)


API가 한번에 결과를 다 내놓을 때까지 기다리게 하는 것보다, 점진적으로 출력이 보여지면 사용자로 하여금 응답이 나오고 있음을 인지하게 되고 기다릴 필요 없이 응답을 읽을 수 있어 사용자 경험이 향상됩니다.

스트리밍을 사용하려면 FoundationModels에서 제공하는 PartiallyGenerated 타입을 사용하면 됩니다.

```swift
@Observable
@MainActor
final class ItineraryPlanner {
    private var session: LanguageModelSession
    private(set) var itinerary: Itinerary.PartiallyGenerated?

    // ...
    
    func suggestItinerary(dayCount: Int) async throws {
        let stream = session.streamResponse(generating: Itinerary.self) {
            "Generate a \(dayCount)-day itinerary to \(landmark.name)."
                
            "Give it a fun title and description."
                
            "Here is an example, but don't copy it:"
            Itinerary.exampleTripToJapan
        }
    
        for try await partialResponse in stream {
            itinerary = partialResponse
        }
    }
}

struct ItineraryView: View {
    let landmark: Landmark
    let itinerary: Itinerary.PartiallyGenerated

    var body: some View {
        VStack(alignment: .leading) {
            if let title = itinerary.title {
                Text(title)
                    .contentTransition(.opacity)
                    .font(.largeTitle)
                    .fontWeight(.bold)
            }
                
            if let description = itinerary.description {
                Text(description)
                    .contentTransition(.opacity)
                    .font(.subheadline)
                    .foregroundStyle(.secondary)
            }

            // ...
        }
        .animation(.easeOut, value: itinerary)
    }
}
```

### 성능 최적화 API 제공

XCode Instruments에 Foundation Models Instruments를 사용할 수 있습니다.
이 도구는 모델 로딩 시간(Asset Loading), 추론 시간(Inference), 도구 호출에 소요된 시간(Tool Caling)을 시각적으로 분석하여 병목의 원인을 파악하는데 도움을 줍니다.

#### 세션 Prewarming

아래는 모델을 로딩하는데 발생하는 병목을 Instruments를 통해 분석한 예시입니다. 

![](/assets/img/post/2025/084.png)

요청을 보낼 때 모델을 불러오게 되면 모델 로딩 시간만큼 지연이 발생합니다.
이를 해결하기 위해 세션 Prewarming을 사용할 수 있습니다.

![](/assets/img/post/2025/085.png)

사용자가 요청을 하기전에 모델을 미리 로드하여 초기 지연 시간을 줄입니다.
예를 들어, 사용자가 텍스트 필드에 입력을 시작하거나 특정 뷰를 탭하여 모델을 사용할 가능성이 높은 시점에 모델을 예열(Prewarming)할 수 있습니다.

![](/assets/img/post/2025/086.png)

#### 토큰 절약 하기 - IncludeSchemaInPrompt

![](/assets/img/post/2025/087.png)

모델이 이미 응답 형식에 대해 이해를 가지고 있는 경우 (대화 형식 또는 Instruction에 스키마에 대한 전체 예시가 포함된 경우 등) IncludeSchemaInPrompt 속성을 false로 지정하여 프롬프트에 Generable 스키마를 삽입하는 것을 피할 수 있습니다.

이는 토큰 수를 줄이고 총 응답시간을 줄일 수 있습니다.


![](/assets/img/post/2025/088.png)

## 마치며

FoundationModels는 앞으로 iOS 앱 개발에서 점점 더 중요한 역할을 하게 될 것이라 생각합니다.

저는 AI의 미래가 **결국 개인화된 경험**으로 향할 것이라고 생각하는데요.  그런 관점에서 애플이 공개한 FoundationModels 은 AI 생태계에 있어 설득력 있는 방향이라고 느꼈습니다.

iOS 개발자로서 이 프레임워크를 적극 활용해 사용자들에게 개인화된 AI 경험을 제공할 수 있게 되면 좋겠습니다.

## 참고

https://developer.apple.com/kr/videos/play/wwdc2025/259/

https://developer.apple.com/documentation/foundationmodels