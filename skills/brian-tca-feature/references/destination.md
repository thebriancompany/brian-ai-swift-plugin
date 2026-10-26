# Destination 구성

한 번에 하나만 표시할 여러 화면·Alert·ConfirmationDialog를 Destination enum으로 묶을 때 읽는다. [화면 표시](presentation.md)의 PresentationAction 처리 원칙과 [자식 Feature 연결](child-features.md)의 역할별 메서드 분리를 함께 적용한다.

기준은 TCA 1.26.2다. 예제의 자식 Feature와 View는 별도로 구현되어 있다고 가정하며, 전체 예제 컴파일은 검증하지 않았다.

도입 여부는 [화면 표시의 선택 기준](presentation.md)을 따른다. 표시 대상이 하나면 개별 @Presents를 사용하고, 같은 부모가 관리하는 상호 배타적인 표시 대상이 두 개 이상이면 Destination으로 전환한다. 처음부터 해당 복수 화면 요구사항이 확정되어 있으면 Destination으로 시작한다. 중첩 화면은 해당 자식 Feature에서 관리한다.

## 선언과 연결

- **강도:** 조건부 필수
- **적용 조건:** 같은 부모가 관리하는 상호 배타적인 표시 대상이 두 개 이상일 때
- **규칙:** 부모 Feature 내부의 State·Action 다음에 `@Reducer public enum Destination`을 선언하고, 화면 case는 이름순으로 정렬한다. Path도 있으면 Destination 다음에 두고, CancelID가 필요하면 Path 등 화면 타입 선언 다음에 두고 Dependency는 그 뒤에 배치한다.
- **State:** `@Presents var destination: Destination.State?`로 선언한다. 매크로가 생성한 State에 대한 `extension ParentFeature.Destination.State: Equatable {}`은 부모 Feature와 같은 파일 하단에 둔다. 자식 State도 Equatable을 준수해야 한다.
- **Action:** 기존 `@CasePathable Action.Child` 안에 `case destination(PresentationAction<Destination.Action>)`을 선언한다.
- **연결:** `.ifLet(\.$destination, action: \.child.destination)`을 사용한다. @Reducer enum의 자동 reducer 추론을 사용하여 자식 reducer 클로저는 생략한다. Scope도 함께 있으면 기존 CombineReducers 규칙에 따라 바깥에 ifLet을 붙인다.
- **Do:** 화면을 열 때 `state.destination = .editor(EditorFeature.State())`처럼 case를 지정하고 닫을 때 nil로 만든다.
- **Don't:** Destination용 Action을 최상위 Action에 직접 두거나, 동일한 표시 대상을 별도의 @Presents 프로퍼티와 중복 관리하지 않는다. Destination.State 준수를 별도 파일로 분리하지 않는다.
- **이유:** 서로 배타적인 화면을 하나의 표시 상태로 표현하고 기존 Child Action 구조를 유지한다.
- **예외:** 단일 화면이나 독립적으로 관리해야 하는 상태까지 무조건 이 enum으로 통합하지 않는다. 스택 화면 전환은 [스택 화면 전환](stack-navigation.md)을 따른다.
- **근거:** 사용자의 Destination 선언 위치, Equatable extension 위치 및 기존 Child 구조 유지 결정
- **API 근거:** [기준 커밋의 Tree-based navigation](https://github.com/pointfreeco/swift-composable-architecture/blob/377da4061db10d26337a71bb279c506bb951f50f/Sources/ComposableArchitecture/Documentation.docc/Articles/TreeBasedNavigation.md#enum-state)

## 화면별 이벤트 처리

- **강도:** 필수
- **적용 조건:** Destination의 자식 이벤트를 부모에서 처리할 때
- **규칙:** `child → destination → 표시 대상별 메서드` 순서로 분리한다. 자식 Feature는 그 메서드에서 delegate를 추출한다. destination 메서드는 PresentationAction의 dismiss와 presented를 나누고, presented에서 Destination.Action의 각 화면 case를 명시적으로 분기한다.
- **Do:** 화면별 메서드가 해당 자식 Action을 받아 guard로 delegate를 추출하고 Delegate의 모든 case를 처리한다. 모든 private 처리 메서드는 이름순으로 배치한다.
- **Don't:** Destination이나 Delegate switch에 default 또는 포괄 case를 넣어 새 화면·이벤트의 추가를 숨기지 않는다. 서로 다른 화면의 delegate 처리를 destination 메서드에 모두 넣지 않는다.
- **이유:** 화면 추가와 delegate 추가 시 부모의 처리 위치를 확인할 수 있다.
- **예외:** Delegate가 없는 화면에는 빈 Delegate를 추가하지 않는다. 부모의 추가 처리가 없으면 해당 화면 분기에서 명시적으로 `.none`을 반환한다. Alert·ConfirmationDialog case는 [버튼 이벤트 규칙](alerts.md)에 따라 버튼 Action을 직접 처리하고 가짜 Delegate를 만들지 않는다. dismiss·presented 분리는 destination 메서드에서 한 번 수행한다.
- **근거:** 사용자가 승인한 Destination 분기 및 화면별 delegate 처리 예시

## Do: 부모 Feature

EditorFeature.Action.Delegate는 cancelled·saved, SettingsFeature.Action.Delegate는 closed를 가진다고 가정한다. 두 자식 State는 Equatable을 준수하고 기본 initializer를 제공한다. 이 예제는 부모가 Delegate를 받아 닫는 경우다. 직접 닫을 수 있는 경우의 선택은 [화면 닫기 기준](presentation.md)을 따른다.

```swift
import ComposableArchitecture

@Reducer
public struct ParentFeature {
  @ObservableState
  public struct State: Equatable {
    @Presents var destination: Destination.State?

    public init() {}
  }

  public enum Action: ViewAction {
    case child(Child)
    case view(View)

    @CasePathable
    public enum Child {
      case destination(PresentationAction<Destination.Action>)
    }

    public enum View {
      case editButtonTapped
      case settingsButtonTapped
    }
  }

  @Reducer
  public enum Destination {
    case editor(EditorFeature)
    case settings(SettingsFeature)
  }

  public init() {}

  public var body: some ReducerOf<Self> {
    Reduce { state, action in
      switch action {
      case let .child(action):
        return child(into: &state, action: action)

      case let .view(action):
        return view(into: &state, action: action)
      }
    }
    .ifLet(\.$destination, action: \.child.destination)
  }

  private func child(
    into state: inout State,
    action: Action.Child
  ) -> Effect<Action> {
    switch action {
    case let .destination(action):
      return destination(into: &state, action: action)
    }
  }

  private func destination(
    into state: inout State,
    action: PresentationAction<Destination.Action>
  ) -> Effect<Action> {
    switch action {
    case .dismiss:
      return .none

    case let .presented(action):
      switch action {
      case let .editor(action):
        return editor(into: &state, action: action)

      case let .settings(action):
        return settings(into: &state, action: action)
      }
    }
  }

  private func editor(
    into state: inout State,
    action: EditorFeature.Action
  ) -> Effect<Action> {
    guard case let .delegate(action) = action else {
      return .none
    }

    switch action {
    case .cancelled:
      state.destination = nil
      return .none

    case .saved:
      state.destination = nil
      return .none
    }
  }

  private func settings(
    into state: inout State,
    action: SettingsFeature.Action
  ) -> Effect<Action> {
    guard case let .delegate(action) = action else {
      return .none
    }

    switch action {
    case .closed:
      state.destination = nil
      return .none
    }
  }

  private func view(
    into state: inout State,
    action: Action.View
  ) -> Effect<Action> {
    switch action {
    case .editButtonTapped:
      state.destination = .editor(EditorFeature.State())
      return .none

    case .settingsButtonTapped:
      state.destination = .settings(SettingsFeature.State())
      return .none
    }
  }
}

extension ParentFeature.Destination.State: Equatable {}
```

## Do: View에서 화면 case 선택

별도 View 파일 예시다. @ViewAction과 send를 사용하며, destination을 scope한 바인딩에서 editor·settings case를 선택한다. sheet 클로저의 Store 이름은 store로 통일한다.

```swift
import ComposableArchitecture
import SwiftUI

@ViewAction(for: ParentFeature.self)
struct ParentView: View {
  @Bindable var store: StoreOf<ParentFeature>

  var body: some View {
    VStack {
      Button("편집") {
        send(.editButtonTapped)
      }

      Button("설정") {
        send(.settingsButtonTapped)
      }
    }
    .sheet(
      item: $store.scope(
        state: \.destination,
        action: \.child.destination
      ).editor
    ) { store in
      EditorView(store: store)
    }
    .sheet(
      item: $store.scope(
        state: \.destination,
        action: \.child.destination
      ).settings
    ) { store in
      SettingsView(store: store)
    }
  }
}
```

## Don't: Child를 우회한 Action과 포괄 분기

```swift
// 최상위 Action 내부: destination은 Action.Child에 둔다.
case destination(PresentationAction<Destination.Action>)

// Destination.Action switch 내부: 새로운 화면의 처리가 숨겨진다.
case let .editor(action):
  return editor(into: &state, action: action)
default:
  return .none
```
