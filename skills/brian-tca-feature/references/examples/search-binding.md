# 검색어 바인딩과 debounce

TextField 검색어 바인딩에 API 호출을 연결할 때 사용하는 예제다. [바인딩](../bindings.md) 및 [Effect](../effects.md) 규칙을 적용한다. 기준은 TCA 1.26.2이며 컴파일은 검증하지 않았다.

User는 Equatable·Sendable을 준수하는 public 타입이고, UserClient는 타입 기반 Dependency로 등록되어 `searchUsers(query:) async throws -> [User]`를 제공한다고 가정한다.

## 예제의 화면별 정책

- 검색어의 양끝 공백은 API 입력에서만 제거하고 State.query의 원래 입력은 유지한다.
- 빈 검색어는 로딩과 결과를 정리한 뒤 진행 중인 대기·요청을 취소한다.
- debounce 대기를 로딩에 포함하고, 새 검색 중에는 기존 결과를 유지한다.
- sleep의 300ms는 변경 가능한 초기 예시 값이다.

위 동작은 이 화면의 가정이며, 모든 검색 화면의 필수 정책으로 확대하지 않는다.

## Feature

```swift
import ComposableArchitecture
import Foundation

@Reducer
public struct SearchFeature {
  @ObservableState
  public struct State: Equatable {
    var errorMessage: String?
    var isLoading = false
    var query = ""
    var users: [User] = []

    public init() {}
  }

  public enum Action: ViewAction {
    case local(Local)
    case view(View)

    public enum Local {
      case usersResponse(Result<[User], any Error>)
    }

    public enum View: BindableAction {
      case binding(BindingAction<State>)
    }
  }

  private enum CancelID: Hashable, Sendable {
    case search
  }

  @Dependency(\.continuousClock) private var clock
  @Dependency(UserClient.self) private var userClient

  public init() {}

  public var body: some ReducerOf<Self> {
    BindingReducer(action: \.view)

    Reduce { state, action in
      switch action {
      case let .local(action):
        return local(into: &state, action: action)

      case let .view(action):
        return view(into: &state, action: action)
      }
    }
  }

  private func local(
    into state: inout State,
    action: Action.Local
  ) -> Effect<Action> {
    switch action {
    case .usersResponse(.failure):
      state.errorMessage = "검색 결과를 불러오지 못했어요."
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
    case .binding(\.query):
      state.errorMessage = nil

      let query = state.query.trimmingCharacters(
        in: .whitespacesAndNewlines
      )

      guard !query.isEmpty else {
        state.isLoading = false
        state.users = []
        return .cancel(id: CancelID.search)
      }

      state.isLoading = true

      return .run { [clock, query, userClient] send in
        try await clock.sleep(for: .milliseconds(300))

        await send(.local(.usersResponse(Result {
          try await userClient.searchUsers(query: query)
        })))
      }
      .cancellable(id: CancelID.search, cancelInFlight: true)

    case .binding:
      return .none
    }
  }
}
```

## View

```swift
import ComposableArchitecture
import SwiftUI

@ViewAction(for: SearchFeature.self)
struct SearchView: View {
  @Bindable var store: StoreOf<SearchFeature>

  var body: some View {
    TextField("검색어", text: $store.query)
  }
}
```

처리 순서는 `$store.query 변경 → BindingReducer의 State 갱신 → view의 binding 분기 → debounce·API 호출 → Local 응답 처리`다.

## Don't: onChange에서 같은 검색을 다시 시작

```swift
// 이미 .binding(\.query)에서 검색을 시작하는 상황의 잘못된 추가 코드.
// searchRequested는 중복 실행을 보여주기 위한 가상의 View Action이다.
TextField("검색어", text: $store.query)
  .onChange(of: store.query) { _, _ in
    send(.searchRequested)
  }
```

검색을 시작하기 위한 onChange와 중계 Action을 추가하지 않는다. API 실행, 취소 및 오류 처리는 Feature가 담당한다.
