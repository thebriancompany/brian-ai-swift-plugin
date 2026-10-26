# 스택 화면 전환

NavigationStack을 기존 Action.Child 구조에 연결할 때 읽는다. 기준은 TCA 1.26.2이며 예제는 자식 Feature·View가 별도로 구현되어 있다고 가정한다. 전체 예제의 컴파일은 검증하지 않았다.

## Path 선언과 연결

- **강도:** 필수
- **적용 조건:** NavigationStack으로 화면을 쌓는 Feature를 작성하거나 수정할 때
- **규칙:** `@Reducer public enum Path`는 부모 Feature 내부에 선언한다. `State → Action → Destination → Path → CancelID → Dependency → init → body → 처리 메서드` 순서를 따르며 필요 없는 선언은 생략한다. Path case는 이름순으로 배치한다.
- **State:** `var path = StackState<Path.State>()`로 시작한다. 매크로가 생성한 State의 `extension ParentFeature.Path.State: Equatable {}`은 같은 파일 하단에 둔다. 각 자식 State도 Equatable을 준수한다.
- **Action:** `@CasePathable Action.Child` 안에 `case path(StackActionOf<Path>)`를 선언한다. StackActionOf는 StackAction<Path.State, Path.Action>의 축약형이다.
- **연결:** `.forEach(\.path, action: \.child.path)`를 사용한다. @Reducer enum의 자동 reducer 추론으로 자식 reducer 클로저는 생략한다. Scope도 함께 있으면 [자식 Feature 연결](child-features.md)에 따라 Scope → Reduce를 CombineReducers로 묶고 바깥에 forEach를 적용한다.
- **Do:** UI 이벤트를 View Action으로 받아 부모에서 `state.path.append(.editor(EditorFeature.State()))`로 push한다.
- **Don't:** Path를 부모 밖의 ParentPath 타입으로 분리하거나 path Action을 최상위 Action에 직접 두지 않는다. StackState 대신 PresentationAction·@Presents로 스택을 모델링하지 않는다.
- **이유:** 기존 Feature 내부 배치와 Child Action 분류를 유지하면서 순서와 요소 ID를 가진 화면 스택을 관리한다.
- **예외:** 한 화면의 모달 표시는 [화면 표시](presentation.md), 상호 배타적인 복수 표시 대상은 [Destination 구성](destination.md)을 따른다. Path는 화면 종류가 하나여도 스택에 필요하면 선언한다.
- **근거:** 사용자의 부모 내부 Path, 같은 파일 하단 Equatable extension 및 Child Action 구조 결정
- **API 근거:** [기준 커밋의 스택 안내](https://github.com/pointfreeco/swift-composable-architecture/blob/377da4061db10d26337a71bb279c506bb951f50f/Sources/ComposableArchitecture/Documentation.docc/Articles/StackBasedNavigation.md)

## 화면별 delegate 처리

- **강도:** 필수
- **적용 조건:** 스택의 자식 이벤트에 부모가 반응할 때
- **규칙:** `child → path → 화면별 메서드 → delegate switch`로 분리한다. path 메서드는 StackAction의 element·popFrom·push를 명시적으로 구분하고 element에서 Path.Action의 각 화면을 분기한다.
- **Do:** element의 StackElementID를 화면별 메서드에 전달하고, 그 메서드에서 guard로 delegate를 추출한다. 화면과 Delegate switch에는 모든 case를 명시한다. private 메서드는 이름순으로 배치한다.
- **Don't:** 화면·Delegate switch의 default나 포괄 case로 추가 이벤트를 숨기지 않는다. 스택 요소 ID를 도메인 모델 ID나 배열 인덱스로 대체하지 않는다.
- **pop 처리:** 이벤트를 보낸 화면부터 닫을 때 `state.path.pop(from: id)`를 사용한다. 해당 화면과 그 위의 화면을 모두 제거한다. 단순히 마지막 화면만 제거하는 popLast와 혼동하지 않는다.
- **push·popFrom 처리:** 부모의 추가 작업이 없으면 각 case에서 `.none`을 반환한다. forEach가 수행하는 push·popFrom의 스택 변경을 부모에서 중복 실행하지 않는다.
- **이유:** 같은 화면 종류가 여러 번 쌓여도 정확한 요소를 처리하고, 새로운 화면·delegate의 누락을 컴파일 시 발견한다.
- **예외:** Delegate가 없는 자식에는 빈 Delegate를 만들지 않는다. 부모의 반응이 없으면 해당 화면 분기에 명시적으로 `.none`을 반환한다.
- **근거:** 사용자가 승인한 스택 ID 전달과 화면별 delegate 처리 예제
- **API 근거:** [기준 커밋의 StackState·StackAction](https://github.com/pointfreeco/swift-composable-architecture/blob/377da4061db10d26337a71bb279c506bb951f50f/Sources/ComposableArchitecture/Reducer/Reducers/StackReducer.swift)

## Do: 부모 Feature

EditorFeature.State는 Equatable을 준수하고 기본 initializer를 제공한다. EditorFeature.Action.Delegate에는 cancelled와 saved가 있다고 가정한다. 부모가 닫기를 결정하는 예제이며, 직접 닫을 수 있는 경우의 선택은 [화면 닫기 기준](presentation.md)을 따른다.

```swift
import ComposableArchitecture

@Reducer
public struct ParentFeature {
  @ObservableState
  public struct State: Equatable {
    var path = StackState<Path.State>()

    public init() {}
  }

  public enum Action: ViewAction {
    case child(Child)
    case view(View)

    @CasePathable
    public enum Child {
      case path(StackActionOf<Path>)
    }

    public enum View {
      case editButtonTapped
    }
  }

  @Reducer
  public enum Path {
    case editor(EditorFeature)
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
    .forEach(\.path, action: \.child.path)
  }

  private func child(
    into state: inout State,
    action: Action.Child
  ) -> Effect<Action> {
    switch action {
    case let .path(action):
      return path(into: &state, action: action)
    }
  }

  private func editor(
    into state: inout State,
    id: StackElementID,
    action: EditorFeature.Action
  ) -> Effect<Action> {
    guard case let .delegate(action) = action else {
      return .none
    }

    switch action {
    case .cancelled:
      state.path.pop(from: id)
      return .none

    case .saved:
      state.path.pop(from: id)
      return .none
    }
  }

  private func path(
    into state: inout State,
    action: StackActionOf<Path>
  ) -> Effect<Action> {
    switch action {
    case let .element(id, action):
      switch action {
      case let .editor(action):
        return editor(into: &state, id: id, action: action)
      }

    case .popFrom:
      return .none

    case .push:
      return .none
    }
  }

  private func view(
    into state: inout State,
    action: Action.View
  ) -> Effect<Action> {
    switch action {
    case .editButtonTapped:
      state.path.append(.editor(EditorFeature.State()))
      return .none
    }
  }
}

extension ParentFeature.Path.State: Equatable {}
```

## Do: NavigationStack View

기존 ViewAction 컨벤션을 유지한다. 스택 바인딩은 Child.path로 scope하고, destination 클로저와 화면별 Store 이름은 store로 사용한다. store.case의 모든 화면을 명시적으로 처리한다.

```swift
import ComposableArchitecture
import SwiftUI

@ViewAction(for: ParentFeature.self)
struct ParentView: View {
  @Bindable var store: StoreOf<ParentFeature>

  var body: some View {
    NavigationStack(
      path: $store.scope(
        state: \.path,
        action: \.child.path
      )
    ) {
      Button("편집") {
        send(.editButtonTapped)
      }
    } destination: { store in
      switch store.case {
      case let .editor(store):
        EditorView(store: store)
      }
    }
  }
}
```

## Don't: 이벤트 ID를 무시하거나 스택 변경을 중복 수행

```swift
// 스택 중간의 화면에서 온 이벤트일 수도 있다.
case .saved:
  state.path.popLast()
  return .none

// StackAction.push는 forEach가 처리한다. 여기서 다시 추가하지 않는다.
case let .push(_, childState):
  state.path.append(childState)
  return .none
```
