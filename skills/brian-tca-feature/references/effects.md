# 비동기 작업과 Effect

API 호출과 결과를 Local Action으로 처리할 때 읽는다. 기준은 TCA 1.26.2다. 아래 코드는 사용자가 선택한 형태의 발췌이며 컴파일은 검증하지 않았다.

## 호출과 결과 전달

- **강도:** 기본값
- **적용 조건:** 성공·실패 결과가 있는 비동기 API 요청을 처리할 때
- **규칙:** `.run` 안에서 필요한 Dependency를 캡처하고 `Result { try await … }`로 성공·실패를 묶어 `await send(.local(...))`로 전달한다. Local Action은 `Result<성공값, any Error>`를 받는다.
- **Dependency:** 타입 기반 의존성으로 등록된 클라이언트는 `@Dependency(UserClient.self) private var userClient` 형태로 선언한다. 위치·접근 제어·정렬은 [개인 컨벤션](conventions.md)을 따른다.
- **Do:** 요청 시작 시 필요한 로딩·오류 상태를 동기적으로 변경한다. 응답 상태는 local 메서드에서 성공·실패로 분기하여 변경한다. 입력 State 값이 필요하면 요청 시점의 값으로 캡처한다.
- **Don't:** 일반적인 단일 요청의 성공·실패 전달을 별도 `.run` catch 클로저로 나누거나 State를 run 안에서 직접 변경하지 않는다. API 호출을 별도의 독립 Task로 감싸 Effect 수명에서 분리하지 않는다.
- **이유:** 성공·실패를 동일한 Local 응답 Action으로 모으고 API 실행과 상태 변경의 역할을 분리한다.
- **예외:** 취소·재시도·스트림 등 별도의 동작이 필요한 경우 이 단일 응답 패턴만으로 정책을 추론하지 않는다. 최신 요청 우선과 명시적 취소는 아래 규칙을 따른다. 재시도·스트림 등의 추가 정책은 별도로 정한다.
- **근거:** 사용자가 제공한 타입 기반 Dependency와 Result 기반 run 예시

## Do: Local Action과 결과 처리

User는 Equatable·Sendable을 준수하는 public 타입이고, UserClient는 타입 기반 Dependency로 접근 가능하며 `fetchUsers() async throws -> [User]`를 제공한다고 가정한다. State에는 `errorMessage: String?`, `isLoading = false`, `users: [User] = []`가 있다. Feature 내부에 아래 선언과 메서드를 배치하고, body의 Reduce는 기존 역할별 메서드 분기 규칙을 따른다.

```swift
public enum Action: ViewAction {
  case local(Local)
  case view(View)

  public enum Local {
    case usersResponse(Result<[User], any Error>)
  }

  public enum View {
    case cancelButtonTapped
    case refreshButtonTapped
  }
}

private enum CancelID: Hashable, Sendable {
  case fetchUsers
}

@Dependency(UserClient.self) private var userClient
```

```swift
private func local(
  into state: inout State,
  action: Action.Local
) -> Effect<Action> {
  switch action {
  case .usersResponse(.failure):
    state.errorMessage = "사용자 목록을 불러오지 못했어요."
    state.isLoading = false
    return .none

  case let .usersResponse(.success(users)):
    state.isLoading = false
    state.users = users
    return .none
  }
}

private func view(
  into state: inout State,
  action: Action.View
) -> Effect<Action> {
  switch action {
  case .cancelButtonTapped:
    state.isLoading = false
    return .cancel(id: CancelID.fetchUsers)

  case .refreshButtonTapped:
    state.errorMessage = nil
    state.isLoading = true

    return .run { [userClient] send in
      await send(.local(.usersResponse(Result {
        try await userClient.fetchUsers()
      })))
    }
    .cancellable(id: CancelID.fetchUsers, cancelInFlight: true)
  }
}
```

이 예시는 최신 요청 우선 정책이다. 로딩 중에도 새 요청을 받아 이전 요청을 대체한다. cancelButtonTapped는 실제 취소 UI가 있는 경우에만 선언한다. 오류 문구와 `usersResponse` 이름도 예시이며 오류 UI·명명 정책 전체를 확정하는 규칙은 아니다.

## Don't: 기본 응답 패턴을 성공과 catch로 분산

```swift
return .run { [userClient] send in
  let users = try await userClient.fetchUsers()
  await send(.local(.usersResponse(.success(users))))
} catch: { error, send in
  await send(.local(.usersResponse(.failure(error))))
}
```

위 문법 자체가 잘못된 것은 아니다. 이 스킬의 기본 선택은 사용자가 지정한 Result 기반 전달 방식이다.

## CancelID와 최신 요청 우선

- **강도:** 조건부 필수
- **적용 조건:** 새 요청이 이전 요청을 대체해야 하는 조회·검색 등의 작업
- **선언:** 부모 Feature 내부에 `private enum CancelID: Hashable, Sendable`을 선언한다. State·Action 및 필요한 Destination·Path 다음, Dependency 앞에 둔다. 실제 취소할 작업만 이름순으로 case를 선언한다.
- **규칙:** 같은 작업에는 같은 CancelID를 사용하고 `.cancellable(id: CancelID.fetchUsers, cancelInFlight: true)`를 run 뒤에 한 줄로 붙인다. 서로 독립적으로 실행해야 하는 작업에는 구분되는 ID를 사용한다.
- **Do:** 새 요청 시작 시 오류 상태를 초기화하고 로딩 상태를 유지한다. 새 요청 결과를 Local Action에서 처리하며 로딩을 해제한다.
- **Don't:** 최신 요청 우선 작업에 `guard !state.isLoading`을 넣어 새 요청을 차단하지 않는다. 서로 무관한 요청에 하나의 ID를 공유해 의도치 않게 취소하지 않는다.
- **이유:** 새 요청이 같은 취소 ID의 진행 중 Effect를 취소하고 대체하게 한다.
- **예외:** 모든 API 호출에 자동 적용하지 않는다. 저장 등 이전 작업의 완료가 중요한 요청은 해당 요구사항에 맞춰 정책을 정한다.
- **근거:** 사용자가 승인한 CancelID 위치와 cancelInFlight: true 및 한 줄 표기

## 명시적 취소

- **강도:** 조건부 필수
- **적용 조건:** 취소 버튼 등 명시적인 이벤트로 진행 중 작업을 취소할 때
- **규칙:** 해당 이벤트를 처리하는 메서드에서 로딩 등 필요한 상태를 먼저 정리한 뒤 `.cancel(id: CancelID.fetchUsers)`를 반환한다.
- **Do:** 예시처럼 `state.isLoading = false`를 설정하고 취소 Effect를 반환한다. 정리할 다른 상태가 있으면 그 작업의 의미에 맞게 함께 갱신한다.
- **Don't:** 취소된 요청의 응답 Action이 도착해야만 로딩이 해제되는 구조를 만들지 않는다. 취소를 일반 API 실패로 표시하기 위한 실패 Action을 임의로 추가하지 않는다.
- **이유:** 작업 취소 시 응답이 전달되지 않아도 UI 상태가 정리되어야 한다.
- **예외:** 새 요청으로 대체하는 경우에는 최신 요청이 진행 중이므로 로딩 상태를 중간에 해제하지 않는다.
- **근거:** 사용자의 상태 정리 후 cancel 반환 결정

## 취소 동작의 범위

TCA 1.26.2의 cancellable은 같은 ID의 진행 중 작업을 취소하며, 취소된 run 작업의 Send는 Action 전달을 무시한다. 이 규칙은 run이 제공한 send와 해당 작업 수명 안에서 실행되는 코드에 적용한다. 이미 전달된 결과를 취소가 되돌리는 것은 아니다.

Result는 오류를 값으로 포착한다. 이를 run의 별도 catch 처리와 동일하다고 설명하지 않는다. 실제 네트워크 작업의 중단 여부는 클라이언트의 취소 지원에 달려 있다.

근거: [기준 커밋의 Cancellation](https://github.com/pointfreeco/swift-composable-architecture/blob/377da4061db10d26337a71bb279c506bb951f50f/Sources/ComposableArchitecture/Effects/Cancellation.swift), [기준 커밋의 Effect·Send](https://github.com/pointfreeco/swift-composable-architecture/blob/377da4061db10d26337a71bb279c506bb951f50f/Sources/ComposableArchitecture/Effect.swift).

### Don't: 새 요청 차단 또는 취소 상태 방치

```swift
// 최신 요청 우선 정책에서는 새 요청이 들어와야 이전 요청을 대체할 수 있다.
guard !state.isLoading else {
  return .none
}

// isLoading이 true인 상태에서 취소만 반환하면 로딩 UI가 남을 수 있다.
case .cancelButtonTapped:
  return .cancel(id: CancelID.fetchUsers)
```

## 검색 입력 debounce

- **강도:** 기본값
- **적용 조건:** 검색 입력이 멈춘 뒤 최신 입력으로 API를 호출해야 할 때
- **규칙:** continuousClock의 `try await clock.sleep(for:)` 뒤에 API 호출을 두고, 대기와 호출 전체를 같은 CancelID의 cancellable로 감싼다. 새 입력은 `cancelInFlight: true`로 이전 대기·요청을 취소한다.
- **대기 시간:** `clock.sleep(for: .milliseconds(300))`처럼 호출부에 직접 작성한다. 300ms는 초기 예시 값이며 고정 요구사항이 아니다. 화면 요구사항에 맞춰 숫자를 변경한다. 이를 위해 debounceDuration 프로퍼티, init 파라미터 또는 별도 설정 Dependency를 기본으로 추가하지 않는다.
- **Do:** 요청 시점의 검색어와 clock·클라이언트를 캡처하고, sleep 이후 API 결과만 Result로 묶어 Local Action에 전달한다.
- **Don't:** sleep을 `try?`로 처리해 취소 후 API 호출로 진행하게 하거나, 모든 화면에 300ms를 강제하지 않는다.
- **이유:** 지연 값은 호출 위치에서 간단히 관리하고, 이전 입력의 대기와 요청은 기존 최신 요청 우선 정책으로 대체한다.
- **예외:** 공백 정규화, 빈 입력 시 결과 초기화, 대기 중 로딩 표시 여부는 화면별 동작으로 정한다. 아래 발췌는 해당 입력·상태 처리를 마친 뒤 반환하는 Effect이며 그 정책을 추가로 확정하지 않는다.
- **근거:** 사용자의 직접 작성 방식 및 필요에 따른 시간 변경 결정, 기존 최신 요청 우선 정책

TextField 바인딩에서 debounce를 시작하는 경우 [검색 바인딩 예제](examples/search-binding.md)를 읽는다. 추가 작업은 view 메서드의 `.binding(\.query)`에서 시작하고 View onChange에서 중복 실행하지 않는다.

### Do: 호출부에 대기 시간 작성

`CancelID.search`와 `@Dependency(\.continuousClock) private var clock`이 선언되어 있고, query는 요청할 검색어라고 가정한다. UserClient는 `searchUsers(query:) async throws -> [User]`를 제공한다.

```swift
return .run { [clock, query, userClient] send in
  try await clock.sleep(for: .milliseconds(300))

  await send(.local(.usersResponse(Result {
    try await userClient.searchUsers(query: query)
  })))
}
.cancellable(id: CancelID.search, cancelInFlight: true)
```

sleep은 Result 바깥에 둔다. 대기 중 취소로 발생한 CancellationError는 run으로 전파되어 작업을 종료하며, API 실패 Local Action으로 바꾸지 않는다. 다음 입력으로 대체하거나 명시적으로 취소할 때의 상태 처리는 위 취소 규칙을 따른다.

### Don't: 기본 패턴에 대기 시간 주입 추가

```swift
private let debounceDuration: Duration

public init(debounceDuration: Duration) {
  self.debounceDuration = debounceDuration
}
```

외부 설정 요구사항이 별도로 없는 경우 위 형태를 도입하지 않는다. 기본 형태는 sleep 호출부에 직접 값을 작성하는 것이다.

## 여러 Effect를 나열하는 형식

`.merge`의 Effect 인자 사이에는 빈 줄을 넣지 않는다. 각 Effect는 필요한 만큼 여러 줄로 작성하고, 이전 Effect의 쉼표 바로 다음 줄에서 다음 Effect를 시작한다. [목록 표기 규칙](conventions.md)을 따른다.

아래는 사용자가 지정한 표기 예시다. NoticeClient·UserClient, 결과 Local Action, 각 CancelID 및 로딩 프로퍼티가 선언되어 있다고 가정한다. 독립적인 두 요청은 각각의 응답 처리에서 해당 로딩 상태를 해제한다.

```swift
private func view(
  into state: inout State,
  action: Action.View
) -> Effect<Action> {
  switch action {
  case .refreshButtonTapped:
    state.isLoadingNotices = true
    state.isLoadingUsers = true

    return .merge(
      .run { [noticeClient] send in
        await send(.local(.noticesResponse(Result {
          try await noticeClient.fetchNotices()
        })))
      }
      .cancellable(id: CancelID.fetchNotices, cancelInFlight: true),
      .run { [userClient] send in
        await send(.local(.usersResponse(Result {
          try await userClient.fetchUsers()
        })))
      }
      .cancellable(id: CancelID.fetchUsers, cancelInFlight: true)
    )
  }
}
```

## Effect 병렬·순차 실행

- **강도:** 필수
- **적용 조건:** 여러 비동기 작업이나 Effect를 함께 처리할 때
- **규칙:** 서로 독립적인 작업은 `.merge`로 실행한다. 순서 또는 결과 의존성이 있는 작업은 하나의 `.run` 안에서 await 순서로 처리하며 `.concatenate`는 사용하지 않는다.
- **Do:** 앞 작업이 성공해야 다음 작업을 수행하는 경우 같은 Result 안에서 `try await`를 순서대로 호출한다. 앞 작업이 오류를 던지면 뒤 작업을 실행하지 않고 실패를 Local Action으로 전달한다.
- **Don't:** `.concatenate`로 작업을 연결하거나, Action을 차례로 send하는 것만으로 해당 Action이 시작한 비동기 Effect의 완료까지 기다린다고 가정하지 않는다. 실행 순서에 의미가 있는 작업을 알파벳순으로 재배치하지 않는다.
- **이유:** 순서와 실패 전파를 한 비동기 흐름에서 명시한다. TCA 1.26.2에서 concatenate는 ComposableArchitecture2Deprecations 활성화 시 deprecated로 표시되며 run 안에서 순차 처리하도록 안내한다. merge에는 해당 deprecation 표시가 없다.
- **예외:** 앞 작업 실패 후에도 다음 작업을 수행해야 하는 요구사항은 별도의 오류 분기로 표현한다. 저장 같은 작업에 최신 요청 우선 취소 정책을 자동 적용하지 않는다.
- **근거:** 사용자의 concatenate 미사용 및 run 순차 실행 선택
- **API 근거:** [기준 커밋의 Effect 조합 구현](https://github.com/pointfreeco/swift-composable-architecture/blob/377da4061db10d26337a71bb279c506bb951f50f/Sources/ComposableArchitecture/Effect.swift)

### Do: 하나의 run에서 순차 호출

캐시 초기화가 성공한 뒤 목록을 조회하는 발췌다. clearUsers는 async throws 함수라고 가정한다.

```swift
return .run { [cacheClient, userClient] send in
  await send(.local(.usersResponse(Result {
    try await cacheClient.clearUsers()
    return try await userClient.fetchUsers()
  })))
}
```

### Don't: concatenate로 순차 작업 구성

```swift
return .concatenate(firstEffect, secondEffect)
```

## 화면 진입 시 데이터 로딩

- **분류:** 개인 컨벤션
- **강도:** 기본값
- **적용 조건:** View의 `.task`에서 화면 데이터를 조회할 때
- **규칙:** `Action.View.task`를 선언하고 `view(into:action:)`의 `.task`에서 요청한다. `hasLoaded`를 추가하거나 `isLoading` guard로 요청을 차단하지 않는다. View에서는 `await send(.task).finish()`로 연결한다.
- **Do:** 요청 시작 시 오류·로딩 상태를 갱신하고, Result 기반 run으로 Local Action에 결과를 전달한다. 같은 조회 작업은 `.cancellable(id: CancelID.fetchUsers, cancelInFlight: true)`로 이전 요청을 대체한다. isLoading은 로딩 UI 표시에 사용한다.
- **Don't:** 화면 진입 조회를 기본적으로 최초 성공 한 번으로 제한하거나, `.task` 안에서 `send(.task)`만 호출해 Effect의 완료 대기와 취소 연결을 생략하지 않는다.
- **이유와 예외:** 재진입 시 다시 요청하며, 진행 중 조회는 새 요청으로 대체한다. `.task`는 전체 생애에서 단 한 번이라는 보장이 없고, finish는 해당 Action에서 시작한 Effect의 수명을 호출한 비동기 작업에 연결한다. 최초 한 번만 조회하거나 화면 이탈 후에도 작업을 유지해야 하는 별도 요구사항은 해당 정책을 명시적으로 정한다.
- **취소와 State:** finish를 통한 취소는 State를 자동 초기화하지 않는다. 취소 후 같은 State에 isLoading이 남아도 다음 `.task` 요청을 막지 않는다. 재요청의 응답에서 로딩을 해제한다. 이탈 즉시 상태 정리가 필요하면 별도 이벤트 정책을 정하며, 명시적 취소 이벤트에서는 기존 상태 정리 후 cancel 규칙을 따른다.
- **근거:** 사용자의 hasLoaded·isLoading guard 미사용 및 `await send(.task).finish()` 결정
- **API 근거:** [기준 커밋의 Store·StoreTask](https://github.com/pointfreeco/swift-composable-architecture/blob/377da4061db10d26337a71bb279c506bb951f50f/Sources/ComposableArchitecture/Store.swift), [ViewActionSending.send](https://github.com/pointfreeco/swift-composable-architecture/blob/377da4061db10d26337a71bb279c506bb951f50f/Sources/ComposableArchitecture/Observation/ViewAction.swift)
- **검토 버전:** TCA 1.26.2. 아래 예제의 컴파일은 검증하지 않았다.

### Do: View task에서 조회 Effect 연결

아래는 Feature 내부 발췌다. 위 호출·결과 처리 예제의 State, Local.usersResponse, CancelID.fetchUsers, userClient와 local 메서드를 사용한다. State의 오류 프로퍼티는 `var errorMessage: String?`로 선언한다. 이 발췌의 Action.View는 `case task`만 가진다. 앞선 버튼 예제의 View enum을 그대로 합치는 것이 아니며, 다른 이벤트도 필요하면 해당 분기를 함께 구현한다. body의 Reduce에서 기존 역할별 메서드로 분기한다. User는 Identifiable을 준수하고 name을 제공하며 View와 State는 같은 모듈에 있다고 가정한다.

```swift
private func view(
  into state: inout State,
  action: Action.View
) -> Effect<Action> {
  switch action {
  case .task:
    state.errorMessage = nil
    state.isLoading = true

    return .run { [userClient] send in
      await send(.local(.usersResponse(Result {
        try await userClient.fetchUsers()
      })))
    }
    .cancellable(id: CancelID.fetchUsers, cancelInFlight: true)
  }
}
```

```swift
import ComposableArchitecture
import SwiftUI

@ViewAction(for: UsersFeature.self)
struct UsersView: View {
  @Bindable var store: StoreOf<UsersFeature>

  init(_ store: StoreOf<UsersFeature>) {
    self.store = store
  }

  var body: some View {
    List(store.users) { user in
      Text(user.name)
    }
    .overlay {
      if store.isLoading {
        ProgressView()
      } else if let errorMessage = store.errorMessage {
        Text(errorMessage)
      }
    }
    .task {
      await send(.task).finish()
    }
  }
}
```

### Don't: 진입 요청 차단 또는 finish 생략

```swift
case .task:
  guard !state.hasLoaded, !state.isLoading else {
    return .none
  }
  // 요청 시작
```

```swift
.task {
  send(.task)
}
```

## 당겨서 새로고침

- **분류:** 개인 컨벤션
- **강도:** 기본값
- **적용 조건:** SwiftUI의 `.refreshable`로 화면 데이터를 새로 조회할 때
- **규칙:** 이벤트는 `Action.View.refresh`로 선언하고 View에서 `await send(.refresh).finish()`로 연결한다. 화면 진입과 같은 동작이면 view 메서드의 `case .refresh, .task:`에서 함께 처리한다.
- **Do:** 같은 조회 작업은 같은 CancelID와 `cancelInFlight: true`를 사용한다. hasLoaded·isLoading guard 없이 요청하고 결과는 기존 Local Action에서 처리한다. 이 예제에서는 요청 중 기존 목록을 유지하고 성공 응답으로 갱신한다.
- **Don't:** refreshable 안에서 send만 호출하거나 별도 Task로 감싸 완료 대기를 끊지 않는다. 동일한 조회 코드를 이벤트별로 복제하지 않는다.
- **이유와 예외:** refreshable의 비동기 클로저가 조회 Effect 완료를 기다리게 한다. 진입과 새로고침의 동작이 다른 경우에는 분기를 나눈다. 목록 유지와 별도 로딩 UI는 화면 요구사항에 맞추며, refreshable의 표시를 위해 별도의 State 플래그를 의무적으로 추가하지 않는다. 취소 시 State 처리는 위 화면 진입·명시적 취소 규칙을 따른다.
- **근거:** 사용자가 승인한 refresh 이벤트, finish 연결 및 동일 조회 분기 통합 예제
- **API 근거:** [기준 커밋의 Store·StoreTask](https://github.com/pointfreeco/swift-composable-architecture/blob/377da4061db10d26337a71bb279c506bb951f50f/Sources/ComposableArchitecture/Store.swift), [Apple refreshable](https://developer.apple.com/documentation/swiftui/view/refreshable(action:))
- **검토 버전:** TCA 1.26.2. 예제 컴파일은 검증하지 않았다.

### Do: 동일한 진입·새로고침 조회

위 화면 진입 예제의 State, CancelID, Dependency와 Local 응답 처리를 사용한다. 아래 Action은 앞선 예제의 대체 선언이다. Action과 view 메서드를 Feature 내부의 각 배치 위치에 두고 body에서 기존 역할별 메서드로 분기한다.

```swift
public enum Action: ViewAction {
  case local(Local)
  case view(View)

  public enum Local {
    case usersResponse(Result<[User], any Error>)
  }

  public enum View {
    case refresh
    case task
  }
}

private func view(
  into state: inout State,
  action: Action.View
) -> Effect<Action> {
  switch action {
  case .refresh, .task:
    state.errorMessage = nil
    state.isLoading = true

    return .run { [userClient] send in
      await send(.local(.usersResponse(Result {
        try await userClient.fetchUsers()
      })))
    }
    .cancellable(id: CancelID.fetchUsers, cancelInFlight: true)
  }
}
```

```swift
import ComposableArchitecture
import SwiftUI

@ViewAction(for: UsersFeature.self)
struct UsersView: View {
  @Bindable var store: StoreOf<UsersFeature>

  init(_ store: StoreOf<UsersFeature>) {
    self.store = store
  }

  var body: some View {
    List(store.users) { user in
      Text(user.name)
    }
    .refreshable {
      await send(.refresh).finish()
    }
    .task {
      await send(.task).finish()
    }
  }
}
```

### Don't: 새로고침 클로저를 먼저 종료

```swift
.refreshable {
  send(.refresh)
}
```

```swift
.refreshable {
  Task {
    await send(.refresh).finish()
  }
}
```
