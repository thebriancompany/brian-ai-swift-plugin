# 양방향 바인딩

TextField·Toggle 등 UI 컨트롤과 State를 양방향으로 연결할 때 읽는다. 기준은 TCA 1.26.2다. 아래는 구조를 설명하는 예제이며 컴파일은 검증하지 않았다.

## 선언과 처리 순서

- **강도:** 조건부 필수
- **적용 조건:** Feature에서 BindingAction 기반 양방향 바인딩이 필요할 때
- **규칙:** 최상위 Action은 ViewAction을 준수하고, 중첩 `Action.View`가 BindableAction을 준수한다. View enum 내부에 `case binding(BindingAction<State>)`를 선언하고 다른 View 이벤트와 이름순으로 정렬한다.
- **Reducer:** `BindingReducer(action: \.view)`를 부모 Reduce 앞에 배치한다. 실제 문법은 `.view`가 아니라 case key path인 `\.view`다. 기존 역할 분기를 유지해 Reduce에서 view 메서드를 호출한다.
- **Do:** BindingReducer가 먼저 변경한 State를 view 메서드에서 읽는다. 추가 반응이 필요한 프로퍼티는 `.binding(\.name)`처럼 구분한다.
- **Don't:** binding case를 최상위 Action이나 Local에 두지 않는다. 이 구조에서 경로 없는 BindingReducer()를 사용하거나, BindingReducer가 이미 반영한 값을 view 메서드에서 반복 대입하지 않는다.
- **이유:** UI 바인딩을 기존 View Action 역할에 포함하고, 후속 처리에서 갱신된 상태를 사용할 수 있게 한다.
- **예외:** 바인딩이 없는 Feature에는 BindableAction·binding case·BindingReducer를 추가하지 않는다. BindingReducer와 Scope가 함께 있으면 둘 다 부모 Reduce 앞에 둔다는 기존 조건을 유지한다. modifier를 조합할 때는 필요한 부모 reducer들을 CombineReducers 안에 포함한다.
- **근거:** 사용자의 Action.View BindableAction 준수와 BindingReducer 배치 결정
- **API 근거:** [기준 커밋의 BindingReducer](https://github.com/pointfreeco/swift-composable-architecture/blob/377da4061db10d26337a71bb279c506bb951f50f/Sources/ComposableArchitecture/Reducer/Reducers/BindingReducer.swift)

## View 연결

- **강도:** 조건부 필수
- **적용 조건:** 위 바인딩 도메인을 SwiftUI 컨트롤에 연결할 때
- **규칙:** View는 `@ViewAction(for: Feature.self)`를 유지하고 Store에 @Bindable을 적용한다. 컨트롤에는 `$store.name`, `$store.isEnabled`처럼 프로퍼티 바인딩을 전달한다.
- **Do:** TextField의 text와 Toggle의 isOn에 `$store.프로퍼티`를 사용한다. 버튼 등 명시적 이벤트는 기존 send 방식을 유지한다.
- **Don't:** 단순 State 바인딩을 위해 수동 Binding(get:set:)이나 직접적인 store.send 호출을 기본으로 추가하지 않는다.
- **이유:** TCA가 변경을 `.view(.binding(...))`으로 전달하도록 연결한다.
- **예외:** 별도 변환이나 검증이 필요한 입력은 요구사항을 확인한다. 이 기본 패턴이 모든 입력 처리 방식을 대체하는 것은 아니다.
- **근거:** 사용자의 `$store.프로퍼티` 연결 결정
- **API 근거:** [기준 커밋의 관찰 기반 바인딩](https://github.com/pointfreeco/swift-composable-architecture/blob/377da4061db10d26337a71bb279c506bb951f50f/Sources/ComposableArchitecture/Observation/Binding%2BObservation.swift)

## Do: Feature

```swift
import ComposableArchitecture

@Reducer
public struct SettingsFeature {
  @ObservableState
  public struct State: Equatable {
    var isEnabled = false
    var name = ""

    public init() {}
  }

  public enum Action: ViewAction {
    case view(View)

    public enum View: BindableAction {
      case binding(BindingAction<State>)
    }
  }

  public init() {}

  public var body: some ReducerOf<Self> {
    BindingReducer(action: \.view)

    Reduce { state, action in
      switch action {
      case let .view(action):
        return view(into: &state, action: action)
      }
    }
  }

  private func view(
    into state: inout State,
    action: Action.View
  ) -> Effect<Action> {
    switch action {
    case .binding:
      return .none
    }
  }
}
```

## Do: View

```swift
import ComposableArchitecture
import SwiftUI

@ViewAction(for: SettingsFeature.self)
struct SettingsView: View {
  @Bindable var store: StoreOf<SettingsFeature>

  init(_ store: StoreOf<SettingsFeature>) {
    self.store = store
  }

  var body: some View {
    Form {
      TextField("이름", text: $store.name)
      Toggle("활성화", isOn: $store.isEnabled)
    }
  }
}
```

## 특정 바인딩에 추가 반응

- **강도:** 조건부 필수
- **적용 조건:** 바인딩으로 변경된 값에 따라 검증·검색 등 추가 작업이 필요할 때
- **규칙:** `view(into:action:)`의 `.binding(\.프로퍼티)` 분기에서 추가 작업을 처리한다. BindingReducer가 반영한 값을 읽고 필요한 Effect를 반환한다. 특정 프로퍼티 분기는 나머지를 받는 `.binding`보다 앞에 둔다.
- **Do:** 검색어 바인딩은 `.binding(\.query)`에서 debounce와 API 호출을 시작한다. 검색 입력을 구현할 때 [검색 바인딩 예제](examples/search-binding.md)와 [Effect 처리](effects.md)를 읽는다.
- **Don't:** 같은 작업을 View의 onChange에서도 실행하지 않는다. 바인딩 후 검색을 시작하기 위해 queryChanged나 searchRequested 같은 중계 Action을 기본으로 추가하지 않는다.
- **이유:** 바인딩 후속 작업의 실행 위치를 reducer 한곳으로 모아 중복 요청을 방지한다.
- **예외:** 이 규칙은 동일한 추가 작업의 중복 실행을 금지한다. 관련 없는 View 전용 onChange까지 일괄 금지하지 않는다. State를 코드에서 직접 변경하는 것은 BindingAction 발생과 같지 않으며, 이 분기가 모든 State 변경을 관찰한다고 가정하지 않는다.
- **근거:** 사용자의 view 메서드 binding 분기 처리 및 View onChange 중복 실행 금지 결정

```swift
switch action {
case .binding(\.name):
  // BindingReducer가 이미 새 이름을 state.name에 반영했다.
  // 이름 변경에 대한 추가 처리만 수행한다.
  return .none

case .binding:
  return .none
}
```

추가 반응이 없다면 특정 프로퍼티 분기를 미리 만들지 않는다. 나머지 binding을 받는 case는 이미 완료된 기본 상태 변경에 추가 작업이 없다는 뜻이다. Delegate의 새 case를 감추는 포괄 처리와 구분하며, Delegate switch의 모든 case를 명시한다는 규칙은 유지한다.

## Don't: BindingReducer 누락 또는 뒤늦은 상태 갱신

```swift
// Don't: view 메서드가 바인딩 이전 State를 읽게 된다.
Reduce { state, action in
  switch action {
  case let .view(action):
    return view(into: &state, action: action)
  }
}
BindingReducer(action: \.view)
```
