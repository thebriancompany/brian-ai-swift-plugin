# Dependency Client 선언과 구현

새 Client를 선언하거나 실제 구현을 등록할 때 읽는다. 스킬 기준은 TCA 1.26.2이며, 아래 예제의 컴파일은 검증하지 않았다. swift-dependencies의 실제 사용 버전은 대상 프로젝트의 의존성 해석 결과를 따른다.

## 선언과 파일 분리

- **분류:** 개인 컨벤션
- **강도:** 기본값
- **적용 조건:** 타입 기반으로 접근할 Dependency Client를 작성할 때
- **규칙:** Client는 `@DependencyClient`와 Sendable을 사용하고, API를 클로저 프로퍼티로 선언한다. 선언은 `UserClient.swift`, DependencyKey 준수와 liveValue 구현은 `UserClient+DependencyKey.swift`에 둔다. 두 파일은 같은 모듈에 둔다.
- **Do:** 외부에서 사용하는 클로저는 public으로 선언한다. 비동기·실패 가능한 API는 `@Sendable (...) async throws -> 결과`로 표현한다. 프로퍼티와 initializer 인자는 이름순으로 정렬한다. import도 이름순으로 정렬한다.
- **Don't:** 파일 분리를 이유로 별도 모듈·타깃을 만들지 않는다. 실제로 동기적이거나 실패하지 않는 API까지 일괄 async throws로 바꾸지 않는다.
- **이유와 예외:** Client 인터페이스와 실제 구현을 구분하면서 모듈 구조는 늘리지 않는다. 기존 프로젝트의 명시적인 모듈 구조는 존중한다.
- **근거:** 사용자의 Client 구성 승인, 파일 분리 및 별도 모듈 미도입 결정
- **검토 버전:** TCA 1.26.2 기준 예제. 컴파일 미검증.

## 등록과 Feature 접근

- **분류:** 개인 컨벤션
- **강도:** 기본값
- **적용 조건:** 위 Client의 실행 시 기본 구현을 연결할 때
- **규칙:** `extension UserClient: DependencyKey`에서 `public static let liveValue`를 제공한다. Feature에서는 `@Dependency(UserClient.self) private var userClient`로 접근한다.
- **Do:** 실제 사용하는 클라이언트만 Feature에 주입하고, 선언 위치와 정렬은 [Dependency 선언](conventions.md)을 따른다.
- **Don't:** 타입 기반 접근만 필요한 상황에 DependencyValues의 키 경로 프로퍼티를 기계적으로 추가하지 않는다.
- **이유와 예외:** 기존 타입 기반 접근 컨벤션을 유지한다. clock·dismiss 등 기존 키 경로 기반 의존성은 해당 API를 그대로 사용한다.
- **근거:** 사용자가 승인한 DependencyKey.liveValue 등록 방식
- **검토 버전:** TCA 1.26.2 기준 예제. 컴파일 미검증.

## 테스트·프리뷰 설정은 필요한 경우에만

- **분류:** 개인 컨벤션
- **강도:** 기본값
- **적용 조건:** Client에 환경별 구현을 추가하려 할 때
- **규칙:** 실제 필요가 없다면 별도의 TestDependencyKey 준수 선언, testValue, previewValue를 추가하지 않는다. 테스트·프리뷰용 파일이나 예제 데이터도 미리 만들지 않는다.
- **Do:** 기본 구조는 DependencyKey와 liveValue로 구성한다. 별도 환경 구현이 요구될 때만 그 범위에서 추가한다.
- **Don't:** 모든 Client에 테스트·프리뷰 구현을 템플릿처럼 생성하지 않는다. 기존에 필요한 구현을 이 규칙만으로 삭제하지 않는다.
- **이유와 예외:** 불필요한 설정을 줄인다. DependencyKey 자체가 TestDependencyKey를 상속하므로 이는 프로토콜 상속을 제거한다는 뜻이 아니라, 별도 선언과 커스텀 값을 기본 생성하지 않는다는 뜻이다. 별도 PreviewKey 프로토콜 대신 previewValue를 사용한다. previewValue를 생략하면 liveValue를 사용하는 기본 동작이 있으므로, 실제 API를 사용하지 않아야 하는 프리뷰는 별도 구현이 필요한 상황이다. 테스트 작성과 예제는 현재 스킬 범위에서 제외한다.
- **근거:** 사용자의 테스트·프리뷰 설정은 필요한 상황에만 사용한다는 결정
- **API 근거:** [DependencyKey 정의](https://github.com/pointfreeco/swift-dependencies/blob/f80552807ec92f72fe3fe4543d71879182b0bfd5/Sources/Dependencies/DependencyKey.swift). TCA 기준 커밋의 Package.resolved에 기록된 swift-dependencies 소스로 확인했으며, 대상 프로젝트의 의존성 버전을 강제하는 규칙은 아니다.

## Do: UserClient.swift

User는 Sendable을 준수하는 public 타입이라고 가정한다. public API가 사용하는 모델 타입도 해당 접근 수준을 만족해야 한다.

```swift
import Dependencies
import DependenciesMacros

@DependencyClient
public struct UserClient: Sendable {
  public var fetchUsers: @Sendable () async throws -> [User]
  public var searchUsers: @Sendable (_ query: String) async throws -> [User]
}
```

이 예제의 `(_ query: String)`에는 매크로가 이름 있는 호출 메서드 `searchUsers(query:)`를 생성한다. 따라서 [검색 바인딩 예제](examples/search-binding.md)의 호출과 같은 인터페이스다. 이는 매크로 동작 설명이며 모든 API에 동일한 인자 이름을 강제하는 규칙은 아니다. [고정 소스의 매크로 설명](https://github.com/pointfreeco/swift-dependencies/blob/f80552807ec92f72fe3fe4543d71879182b0bfd5/Sources/DependenciesMacros/Macros.swift)을 참고한다.

## Do: UserClient+DependencyKey.swift

아래 UserAPI는 프로젝트에 이미 존재하는 실제 API 계층을 가정한 이름이다. 해당 타입의 async throws 정적 메서드가 [User]를 반환한다고 가정하며, 스킬 적용 시 프로젝트의 실제 구현으로 연결한다. 이를 위해 별도의 UserAPI 모듈을 만들라는 의미가 아니다.

```swift
import Dependencies

extension UserClient: DependencyKey {
  public static let liveValue = Self(
    fetchUsers: {
      try await UserAPI.fetchUsers()
    },
    searchUsers: { query in
      try await UserAPI.searchUsers(query: query)
    }
  )
}
```

Feature 내부에서는 다음과 같이 접근한다.

```swift
@Dependency(UserClient.self) private var userClient
```

호출과 Local Action 결과 처리는 [Effect 처리](effects.md)를 따른다.
