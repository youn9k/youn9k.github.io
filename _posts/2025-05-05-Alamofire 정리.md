---
title: "[Swift]Alamofire 정리"
date: 2025-05-05 12:00 +0900
categories:
  - 🍎 iOS
  - Swift
tags:
  - Alamofire
  - 네트워킹
---

Alamofire는 Swift 기반 HTTP 네트워킹 라이브러리로 URLSession을 래핑하여 네트워킹 코드를 쉽고 가독성 좋게 사용할 수 있는 다양한 기능들을 제공합니다.

## URLSession 기반 코드의 문제점

기본적으로 Foundation에서 제공하는 URLSession을 직접 사용할 경우 다음과 같은 단점들이 존재했습니다.
- 요청(Request)을 구성하는 코드가 장황하며, 가독성이 떨어짐
- 반복되는 코드가 발생함(ex: 헤더 설정, 에러 핸들링, JSON 디코딩 등)
- 네트워크 요청을 디버깅하거나 로그를 남기려면 기존 코드에 많은 수정이 필요

아래 예시 코드는 HTTP 요청을 통해  User 정보를 가져오는 간단한 예시입니다.
단순한 GET 요청임에도 불구하고, 에러 처리, 디코딩 등 코드가 복잡해집니다.
```swift
guard let url = URL(string: "https://api.example.com/user") else { return }

var request = URLRequest(url: url)
request.httpMethod = "GET"
request.addValue("application/json", forHTTPHeaderField: "Content-Type")

let task = URLSession.shared.dataTask(with: request) { data, response, error in
    if let error = error {
        print("요청 실패: \(error)")
        return
    }

    guard let data = data else { return }

    do {
        let user = try JSONDecoder().decode(User.self, from: data)
        print("유저 정보: \(user)")
    } catch {
        print("디코딩 실패: \(error)")
    }
}

task.resume()
```

## Alamofire 가 제공하는 기능들

Alamofire는 이러한 불편함을 줄이기 위해 메소드 체이닝 방식의 API 제공, 인/디코딩 메소드 탑재, Interceptor 등의 기능을 제공합니다.

### 메소드 체이닝 방식

요청, 응답 처리, 응답 코드 검증 등을 체이닝 방식으로 작성해 가독성이 좋은 코드를 작성할 수 있습니다.

```swift
AF.request("https://api.example.com/user", method: .get)
    .validate() // default: 200 ~ 299
    .responseDecodable(of: User.self) { response in
        switch response.result {
        case .success(let user):
            print("유저 정보: \(user)")
        case .failure(let error):
            print("요청 실패: \(error)")
        }
    }
```


### JSON 인코딩 & 디코딩 기능 내장

POST요청 시 필요한 Body를 Encodable 객체로 전송하거나, 응답으로 온 JSON 데이터를 Decodable 객체로 디코딩할 수 있습니다.

```swift
struct LoginRequest: Encodable {
    let username: String
    let password: String
}

AF.request("https://api.example.com/login",
           method: .post,
           parameters: LoginRequest(username: "test", password: "1234"),
           encoder: JSONParameterEncoder.default)
    .validate()
    .responseDecodable(of: LoginResponse.self) { response in
        // 응답 처리
    }
```

### 응답 코드에 따른 에러 처리

.validate() 메소드를 통해 200번대의 응답 코드가 아닐 경우 에러로 간주하고 처리할 수 있습니다.

```swift
AF.request("https://api.example.com/resource")
    .validate(statusCode: 200..<300)
    .response { response in
        if let error = response.error {
            print("에러 발생: \(error)")
        }
    }
```

### Interceptor를 활용한 요청 재시도 및 인증 갱신

RequestInterceptor를 구현하여, 요청 시에 필요한 토큰을 추가하거나 반복되는 작업을 수행할 수 있습니다. 또한 Interceptor는 RequestRetrier를 채택하고 있어 요청 실패 시 재시도 또한 구현할 수 있습니다.

```swift
// 프로토콜

public protocol RequestAdapter : Sendable {
    func adapt(_ urlRequest: URLRequest, for session: Alamofire.Session, completion: @escaping @Sendable (_ result: Result<URLRequest, any Error>) -> Void)
    func adapt(_ urlRequest: URLRequest, using state: Alamofire.RequestAdapterState, completion: @escaping @Sendable (_ result: Result<URLRequest, any Error>) -> Void)
}

public protocol RequestRetrier : Sendable {
    func retry(_ request: Alamofire.Request, for session: Alamofire.Session, dueTo error: any Error, completion: @escaping @Sendable (Alamofire.RetryResult) -> Void)
}

public protocol RequestInterceptor : Alamofire.RequestAdapter, Alamofire.RequestRetrier {
}

// 구현부

class AuthInterceptor: RequestInterceptor {
    func adapt(_ urlRequest: URLRequest, for session: Session, completion: @escaping (Result<URLRequest, Error>) -> Void) {
        // 요청에 토큰 추가
        var adaptedRequest = urlRequest
        adaptedRequest.addValue("Bearer \(TokenManager.shared.accessToken)", forHTTPHeaderField: "Authorization")
        completion(.success(adaptedRequest))
    }

    func retry(_ request: Request, for session: Session, dueTo error: Error, completion: @escaping (RetryResult) -> Void) {
        // 토큰 만료 시 재발급 로직 등
    }
}

// 사용 예시

let session = Session(interceptor: AuthInterceptor())

session.request("https://api.example.com/secure-data")
    .validate()
    .responseJSON { response in
        print(response)
    }
```

### AuthenticationInterceptor

Interceptor를 대부분 토큰 기반 인증을 위해 사용하다보니, Alamofire 5.2부터 기본 구현체로 AuthenticationInterceptor을 제공합니다.

AuthenticationCredential과 Authenticator 프로토콜을 준수하는 구현체를 만들고 두 구현체를 주입하여 사용합니다.

### AuthenticationCredential
token값들을 가지고 있고, 만료시간 정보를 보고 refresh가 필요한지 판단하여 requiresRefresh 플래그값에 적용
```swift
public protocol AuthenticationCredential {
    var requiresRefresh: Bool { get }
}

// 구현체
struct AuthToken: AuthenticationCredential {
    let accessToken: String
    let refreshToken: String
    let expiredAt: Date

    // 유효시간이 앞으로 5분 이하 남았다면 refresh가 필요하다고 true를 리턴 (false를 리턴하면 refresh 필요x)
    var requiresRefresh: Bool { Date(timeIntervalSinceNow: 60 * 5) > expiredAt }
}
```

### Authenticator
Credential타입을 가지고, 이 타입의 token을 가지고 refresh하기까지 4단계 수행(apply -> didRequest -> isRequest -> refersh)
```swift
public protocol Authenticator: AnyObject, Sendable {
associatedtype Credential: AuthenticationCredential & Sendable
    func apply(_ credential: Credential, to urlRequest: inout URLRequest)
    func didRequest(_ urlRequest: URLRequest, with response: HTTPURLResponse, failDueToAuthenticationError error: any Error) -> Bool
    func isRequest(_ urlRequest: URLRequest, authenticatedWith credential: Credential) -> Bool
    func refresh(_ credential: Credential, for session: Session, completion: @escaping @Sendable (Result<Credential, any Error>) -> Void)
}

// 구현체
class TokenAuthenticator: Authenticator {
    func apply() { ... }
    func didRequest() { ... }
    func isRequest() { ... }
    func refresh() { ... }
}
```

1. **apply()**
api요청 시 AuthenticatorIndicator객체가 존재하면, 요청 전에 가로채서 apply에서 Header에 bearerToken 추가

```swift
func apply(_ credential: AuthToken, to urlRequest: inout URLRequest) {
    urlRequest.headers.add(.authorization(bearerToken: credential.accessToken))
    urlRequest.addValue(credential.refreshToken, forHTTPHeaderField: "refresh-token")
}
```

2. **didRequest()**
api요청 후 error가 떨어진 경우, 401에러(인증에러)인 경우만 refresh가 되도록 필터링

```swift
func didRequest(_ urlRequest: URLRequest, with response: HTTPURLResponse, failDueToAuthenticationError error: Error) -> Bool {
    return response.statusCode == 401
}

```

3. **isRequest()**
인증이 필요한 urlRequest에 대해서만 refresh가 되도록, 이 경우에만 true를 리턴하여 refresh 요청

```swift
func isRequest(_ urlRequest: URLRequest, authenticatedWith credential: AuthToken) -> Bool {
    // bearerToken의 urlRequest대해서만 refresh를 시도 (true)
    let bearerToken = HTTPHeader.authorization(bearerToken: credential.accessToken).value
    return urlRequest.headers["Authorization"] == bearerToken
}
```

4. **refresh()**
token을 refresh 하는 부분
```swift
func refresh(_ credential: AuthToken, for session: Session, completion: @escaping (Result<AuthToken, Error>) -> Void) {
        let body = RefreshTokenRequest(refreshToken: credential.refreshToken)

        session.request("https://api.example.com/auth/refresh",
                        method: .post,
                        parameters: body,
                        encoder: JSONParameterEncoder.default)
            .validate()
            .responseDecodable(of: TokenResponse.self) { response in
                switch response.result {
                case .success(let tokenResponse):
                    let newCredential = AuthToken(
                        accessToken: tokenResponse.accessToken,
                        refreshToken: tokenResponse.refreshToken,
                        expiration: Date().addingTimeInterval(tokenResponse.expiresIn)
                    )
                    completion(.success(newCredential))

                case .failure(let error):
                    completion(.failure(error))
                }
            }
    }
```


### EventMonitor를 통한 디버깅 및 로깅

Alamofire는 요청-응답 시의 생명주기 이벤트들을 감시할 수 있는 EventMonitor를 제공합니다.
이를 활용해 요청 시작/종료, 응답 수신 등을 추적하고, 디버깅 및 로깅에 활용할 수 있습니다.

```swift
public protocol EventMonitor : Sendable {
    var queue: DispatchQueue { get }
    func urlSession(_ session: URLSession, didBecomeInvalidWithError error: (any Error)?)
    func urlSession(_ session: URLSession, task: URLSessionTask, didReceive challenge: URLAuthenticationChallenge)
    func urlSession(_ session: URLSession, task: URLSessionTask, didSendBodyData bytesSent: Int64, totalBytesSent: Int64, totalBytesExpectedToSend: Int64)
    // ...
    func request(_ request: Alamofire.Request, didCreateInitialURLRequest urlRequest: URLRequest)
    func request(_ request: Alamofire.Request, didFailToCreateURLRequestWithError error: Alamofire.AFError)
    func request(_ request: Alamofire.Request, didAdaptInitialRequest initialRequest: URLRequest, to adaptedRequest: URLRequest)
    // ...
    func requestIsRetrying(_ request: Alamofire.Request)
    func requestDidFinish(_ request: Alamofire.Request)
    func requestDidResume(_ request: Alamofire.Request)
    func requestDidCancel(_ request: Alamofire.Request)
    // ...
}

final class NetworkLogger: EventMonitor {
    let queue = DispatchQueue(label: "com.example.networklogger")

    func requestDidResume(_ request: Request) {
        print("요청 시작: \(request)")
    }

    func request(_ request: DataRequest, didParseResponse response: DataResponse<Data?, AFError>) {
        print("✅ 응답 수신: \(response)")
    }
}

let logger = NetworkLogger()
let session = Session(eventMonitors: [logger])
```

### 네트워크 상태 모니터링
NetworkReachabilityManager 를 통해 네트워크 상태를 감지할 수 있습니다.

```swift
open class NetworkReachabilityManager : @unchecked Sendable {
    public enum NetworkReachabilityStatus : Sendable {
        case unknown
        case notReachable
        case reachable(Alamofire.
        NetworkReachabilityManager.
        NetworkReachabilityStatus.ConnectionType)
        public enum ConnectionType : Sendable {
            case ethernetOrWiFi
            case cellular
        }
    }

    public typealias Listener = @Sendable 
    (Alamofire.
    NetworkReachabilityManager.
    NetworkReachabilityStatus) -> Void
    
    @preconcurrency open func startListening(
    onQueue queue: DispatchQueue = .main, 
    onUpdatePerforming listener: @escaping Alamofire.NetworkReachabilityManager.Listener) -> Bool
    open func stopListening()
}

let reachabilityManager = NetworkReachabilityManager()

reachabilityManager?.startListening { status in
    switch status {
    case .reachable(.ethernetOrWiFi):
        print("WiFi 연결됨")
    case .reachable(.cellular):
        print("셀룰러 연결됨")
    case .notReachable:
        print("인터넷 연결 안됨")
    case .unknown:
        print("상태 알 수 없음")
    }
}
```

### 참고

https://ios-development.tistory.com/732