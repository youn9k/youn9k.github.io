---
title: "[Swift]애플 로그인 서버부터 클라이언트까지Swift + Nest.js + TypeScript"
date: 2025-05-22 12:00 +0900
categories:
  - 🍎 iOS
  - Swift
tags:
  - 애플로그인
  - 인증
---


iOS 앱을 개발할 때 소셜 로그인 기능을 구현한다면, 애플 로그인은 사실상 필수적인 기능입니다. 그렇기에 프로젝트를 할 때마다 애플 로그인을 구현하게 되는데, 매번 '어떻게 구현하더라' 하며 다시 찾아보는 일이 반복되곤 했습니다.

“애플로부터 토큰을 발급받아 서버에 넘기면 서버가 알아서 로그인 또는 회원가입 처리를 진행하고, 그 후 액세스 토큰과 리프레시 토큰을 넘겨 받는다” 정도로만 기억하고 있었고, 로그인 플로우에 대한 명확한 이해가 부족했습니다.

마침 클라이언트부터 백엔드까지 직접 구현해볼 수 있는 기회가 생겼고, 이번 기회에 로그인 플로우를 제대로 정리해야겠다고 생각했습니다.

> iOS 클라이언트는 Swift 로, 서버 코드는 Nest.js + TypeScript로 작성된 점 참고해주세요!

## 사전 작업

Apple Developer - Identifiers - 내 앱 ID - Capabilities - Sign In with Apple 체크

![](/assets/img/post/2025/029.png)

XCode - Project - 앱 타겟 - Sigining&Capailities - Sign in with Apple 추가
(Debug와 Release 2개의 타겟을 운용한다면, 두 타겟 모두 추가해줘야 합니다.)

![](/assets/img/post/2025/030.png)

.entitlements 인증서 위치를 옮겼다면, Build Settings 에서 경로를 수정해주어야 합니다.

![](/assets/img/post/2025/031.png)

## 애플 로그인 플로우

로그인 플로우는 아래와 같습니다.

![](/assets/img/post/2025/032.png)

> 코드를 전부 넣기엔 너무 많아서 흐름을 파악할 수 있을 정도로만 넣었어요.

1. 사용자가 애플 로그인을 요청합니다.

```swift
// iOS 코드 AppleAuthUseCase.swift
import AuthenticationServices

let request = ASAuthorizationAppleIDProvider().createRequest()
request.requestedScopes = [.fullName, .email]
            
let controller = ASAuthorizationController(authorizationRequests: [request])
controller.delegate = self
controller.performRequests()
```


2. 애플 로그인 화면을 통해 이름, 이메일 등 사용자 정보 제공을 동의합니다.

![](/assets/img/post/2025/033.jpeg)

3. 인가 코드(AuthorizationCode)와 identityToken 토큰(JWT)이 담긴 ASAuthorizationAppleIDCredential을 발급받습니다. 

4. identityToken을 담아 서버에 로그인을 요청합니다. (인가 코드를 보내는 방식과 JWT를 보내는 방식이 존재하는데, JWT를 보내는게 서버 로직에서 좀 더 편리합니다.)

```swift
// iOS 코드 AppleAuthUseCase.swift
public func authorizationController(
        controller: ASAuthorizationController,
        didCompleteWithAuthorization authorization: ASAuthorization
    ) {
        guard let credential = authorization.credential as? ASAuthorizationAppleIDCredential,
           let rawData = credential.identityToken else { return }
        let token = String(decoding: rawData, as: UTF8.self)
        
        requestLogin(token: token)
    }
```


5. 토큰의 헤더를 파싱하고, base64로 디코딩해 공개 키의 Key ID를 얻습니다.

```ts
// 서버 코드 auth.service.ts
private headerDecode(token: string): { [key: string]: string } {
    const header = token.split('.')[0];
    return JSON.parse(Buffer.from(header, 'base64').toString());
  }
```

6. Key ID를 통해 서명에 사용된 공개키를 얻습니다.

```ts
// 서버 코드 auth.service.ts
import { JwksClient } from 'jwks-rsa'

this.jwksClient = new JwksClient({
      jwksUri: 'https://appleid.apple.com/auth/keys',
    });
    
const key = await this.jwksClient.getSigningKey(header['kid']);
const publicKey = key.getPublicKey();
```

7. 공개키를 통해 유저가 담아서 보낸 토큰의 무결성을 검증한 뒤 내부 값(payload)
를 읽습니다.

```ts
// 서버 코드 auth.service.ts
const userInfo = await this.jwtService.verifyAsync<OauthAppleDto>(token, {
      publicKey,
      algorithms: ['RS256'],
    });
```

identityToken의 payload 형식은 아래와 같습니다.

[Authenticating users with Sign in with Apple](https://developer.apple.com/documentation/signinwithapple/authenticating-users-with-sign-in-with-apple)

```ts
export class OauthAppleDto {
  iss: string;
  sub: string;
  aud: string;
  iat: number;
  exp: number;
  nonce: string;
  nonce_supported: boolean;
  email: string;
  email_verified: boolean;
  is_private_email: boolean;
}
```


8. payload에 담긴 정보를 통해 유저 데이터를 만듭니다.

```ts
// 서버 코드 auth.service.ts
const user = {
  provider: 'apple',
  identifier: profile.sub,
  email: profile.email,
  name: 'user',
};

return user;
```

9. 로그인을 시도하고, DB에 유저 정보가 없다면 회원가입 로직을 수행합니다.

```ts
// 서버 코드 auth.service.ts
// 위에서 만든 유저데이터를 파라미터로 전달합니다.
async login(oauthUser: OauthUserDto): Promise<OAuthLoginResponseDto> {
    let user: UserDto | null = null;
    user = await this.userService.findByIdentifier(
      oauthUser.identifier,
      true
    );
    
    // 일치하는 유저 정보가 없다면, 회원가입
    if (user == null) {
      user = await this.userService.create(oauthUser);
    }

    ...
}
```

10. 액세스 토큰, 리프레쉬 토큰, 액세스 토큰 만료 시간을 생성합니다.

11~12. 리프레쉬 토큰을 유저 테이블에 저장하고 리턴합니다.

3번 단계에서 인가 코드(Authorization Code)를 전달 받아 Client Secret을 생성하고, AccessToken과 RefreshToken을 발급 받아 사용할 수도 있지만,

추후 다른 플랫폼(Naver, Kakao, Google 등)을 함께 지원할 경우 모든 로그인 방식을 하나의 토큰으로 관리하는 것이 더 편리하다고 판단하여 서버에서 직접 AccessToken과 RefreshToken을 생성 및 관리하는 방식을 사용했습니다.

```ts
// 서버 코드 auth.service.ts
async login(oauthUser: OauthUserDto): Promise<OAuthLoginResponseDto> {
    ...
    // 생략
    
    const accessToken = await this.createAccessToken(user);
    const refreshToken = await this.createRefreshToken(user);
  
    await this.userService.insertRefreshToken(user.id, refreshToken);

    return {
      accessToken: accessToken,
      expiredAt: await this.getTokenExpirationTime(),
      refreshToken: refreshToken,
    };
}
```


13. 키체인에 저장합니다.

키체인을 쉽게 사용할 수 있도록 Class를 만들어 관리했습니다.

> kSec으로 시작하는 옵션들은 keyChain 쿼리에 사용되는 키(Key)들로 kSecClas는 저장할 항목의 타입으로 kSecClassGenericPassword로 설정하여 일반 비밀번호 데이터로 취급 저장함을 뜻합니다.

```swift
// KeyChain.swift
public final class KeyChain: Sendable {
    public static let shared = KeyChain()
    private let service: String = Bundle.main.bundleIdentifier ?? ""
    
    private init() {}

    public func save(type: KeyChainType, value: String) {
        let query: NSDictionary = [
            kSecClass: kSecClassGenericPassword,
            kSecAttrService: service,
            kSecAttrAccount: type.rawValue,
            kSecValueData: value.data(using: .utf8, allowLossyConversion: false) ?? .init()
        ]
        SecItemDelete(query)
        SecItemAdd(query, nil)
    }
    public func load(type: KeyChainType) -> String? { ... }
    public func delete(type: KeyChainType) { ... }
}
```

AuthRepository는 Repository 패턴을 사용해 DataSource를 추상화하며 로그인이나 인증 관련된 로 API를 담당하는 객체입니다.
소셜 로그인 엔드포인트와 파라미터를 갖는 AuthAPI 타입을 만들어 HTTP 요청을 전송합니다.
로그인에 성공하면, 토큰과 만료기간을 키체인에 저장합니다.

```swift
// AuthRepository.swift
public func socialLogin(providerType: ProviderType, token: String) async throws -> SocialLoginResponseDTO {

        // 소셜 로그인 요청
        let body = SocialLoginRequestDTO(...)
        let api = AuthAPI.loginWithOAuth(body)
        let response: SocialLoginResponseDTO = try await networkService.requestWithoutAuth(api)

        // 토큰을 키체인에 저장
        keyChain.save(type: .accessToken, value: response.accessToken)
        keyChain.save(type: .refreshToken, value: response.refreshToken)
        keyChain.save(type: .accessExpiredAt, value: String(response.expiredAt))
        
        networkService.updateCredentials()
        return response
    }
}
```

14. 액세스 토큰을 담아 인증이 필요한 API를 요청할 수 있습니다.

NetworkService는 외부에서 서드파티 라이브러리의 의존성을 제거하고 요청을 API 타입으로 추상화하여 간편하게 사용할 수 있도록 구성했습니다.

인증이 필요한 API일 경우 요청 헤더에 액세스 토큰을 담아야합니다.
request 메소드의 파라미터인 API에 직접 토큰을 담아도 되지만, 휴먼 에러와 보일러 플레이트를 줄이기 위해 Alamofire의 기능인 AuthenticationInterceptor 를 활용했습니다.

```swift
// NetworkService.swift
import Alamofire

public final class NetworkService {
    public static let shared = NetworkService()
    // 생략
    private let networkLogger: NetworkLogger
    private var authSession: Session
    private var defaultSession: Session
    
    private init() {
        // 생략
        let authenticator = OAuthAuthenticator()
        let credential = OAuthCredential(..)
        let intercepter = AuthenticationInterceptor(..)
        let networkLogger = NetworkLogger()
        
        self.authSession = Session(
            interceptor: interceptor,
            eventMonitors: [networkLogger]
        )
        
        self.defaultSession = Session(
            eventMonitors: [networkLogger]
        )
    }

    public func request<T: Decodable>(_ api: API) async throws -> T {
        let urlRequest = try api.asURLRequest()
        
        let task: DataTask<T> = authSession.request(urlRequest)
            .validate()
            .serializingDecodable(T.self)
        
        let response = await task.response
        switch response.result {
        case .success(let data): return data
        case .failure(let error): throw error
        }
    }

    public func requestWithoutAuth<T: Decodable>(_ api: API) async throws -> T {
        let urlRequest = try api.asURLRequest()
        
        let task: DataTask<T> = defaultSession.request(urlRequest)
            .validate()
            .serializingDecodable(T.self)
        
        let response = await task.response
        switch response.result {
        case .success(let data): return data
        case .failure(let error): throw error
        }
    }
}
```

AuthenticationInterceptor는 Authenticator와 Credential 을 주입받아 만들어집니다.
Authenticator는 요청부터 응답 사이에 발생하는 인증 관련 주요 생명주기 메소드를 제공해 OAuth 과정에 필요한 작업들을 쉽게 수행할 수 있도록 도와줍니다. 

apply 를 통해 요청을 보내기 전, 요청 헤더에 액세스 토큰을 추가합니다.

didRequest는 응답의 결과가 에러일 경우, 에러 코드 구분을 통해 다음 생명주기 메소드를 수행할 지 결정합니다. statusCode를 401로 필터링하여 인증 관련 에러일 경우에만 다음 메소드를 수행하도록 합니다.

isRequest는 실패한 요청에서 사용된 토큰을 Credential과 비교하여 해당 토큰의 인증 시도 여부를 체크합니다. 이미 같은 토큰으로 인증된 요청이었다면, 다시 같은 토큰으로 요청해봤자 다시 실패할 것이므로 새로 토큰을 발급받도록 true를 리턴합니다. 

제 코드에선 구현되지 않았지만 실패한 요청에서 사용된 토큰이 Credential과 다르다면 Credential의 토큰을 사용해 재요청을 수행할 수 있습니다.

refresh는 메소드 그대로 토큰을 재발급 받는 로직을 수행하는 메소드입니다. 재발급 로직까지 모두 설명하기엔 글이 너무 길어질 것 같아 생략했습니다.

```swift
final class OAuthAuthenticator: Authenticator {
    private let keyChain = KeyChain.shared
    
    func apply(
        _ credential: OAuthCredential,
        to urlRequest: inout URLRequest
    ) {
        urlRequest.headers.add(.authorization(bearerToken: credential.accessToken))
    }
    
    func didRequest(
        _ urlRequest: URLRequest,
        with response: HTTPURLResponse,
        failDueToAuthenticationError error: any Error
    ) -> Bool {
        return response.statusCode == 401
    }
    
    func isRequest(
        _ urlRequest: URLRequest,
        authenticatedWith credential: OAuthCredential
    ) -> Bool {
        let bearerToken = HTTPHeader.authorization(bearerToken: credential.accessToken).value
        return urlRequest.headers["Authorization"] == bearerToken
    }
    
    func refresh(
        _ credential: OAuthCredential,
        for session: Session,
        completion: @escaping (Result<OAuthCredential, any Error>) -> Void
    ) {
        // RefreshToken을 통해 토큰 재발급 로직 수행
    }
}

```