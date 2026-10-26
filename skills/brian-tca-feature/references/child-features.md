# 자식 Feature 연결

이 문서는 개인 컨벤션이다. 기준은 TCA 1.26.2이며, 예제는 자식 타입이 별도로 구현되어 있다고 가정한 설명용 발췌다. 전체 예제의 컴파일은 검증하지 않았다.

## Child Action과 Scope

- **강도:** 필수
- **적용 조건:** 항상 존재하는 자식 Feature를 부모에 연결할 때
- **규칙:** 부모 State에 자식 State를 두고, 자식 Action은 `@CasePathable`을 적용한 `Action.Child` 안에 자식별 case로 선언한다. `Scope(\.자식프로퍼티, action: \.child.자식case)`로 연결한다.
- **배치:** Scope를 부모의 Reduce 앞에 둔다. 자식의 처리 후 부모가 이벤트에 반응한다. reducer 조합의 실행 순서를 전체 알파벳 정렬로 바꾸지 않는다.
- **Do:** 자식의 상태 변경·Effect 실행은 Scope에 맡기고, 부모의 `child` 메서드에서는 부모에 필요한 추가 처리를 한다.
- **Don't:** `child` 메서드에서 자식 reducer를 직접 호출하거나 Scope와 중복 실행하지 않는다.
- **이유:** 자식의 처리와 부모의 반응을 분리하면서 부모가 자식 처리 이후의 상태를 참조할 수 있게 한다.
- **예외:** 일반 optional·컬렉션 자식에는 아래의 ifLet·forEach 규칙을 적용한다. 화면 표시에 사용하는 @Presents·PresentationAction은 [화면 표시](presentation.md)를 따른다. 스택 화면 전환은 [스택 화면 전환](stack-navigation.md)을 따른다.
- **근거:** 사용자가 승인한 Child Action·Scope 구조와 Scope를 Reduce 앞에 배치하는 결정
- **API 근거:** [기준 커밋의 Scope 구현](https://github.com/pointfreeco/swift-composable-architecture/blob/377da4061db10d26337a71bb279c506bb951f50f/Sources/ComposableArchitecture/Reducer/Reducers/Scope.swift)

## 자식별 메서드 분리와 delegate 처리

- **강도:** 메서드 추가 분리는 조건부, delegate의 명시적 case 처리는 필수
- **적용 조건:** 부모에서 자식 이벤트에 반응할 때. 특히 자식 종류나 처리 코드가 많아질 때 자식별 메서드로 분리할 수 있다.
- **규칙:** `child(into:action:)`는 `Action.Child`를 자식별로 분기한다. 일반 자식과 optional 자식의 분리 메서드는 해당 자식의 Action 타입을 받아 `guard case let .delegate(action) = action`으로 delegate만 추출한다. 이후 switch에서 해당 Delegate enum의 모든 case를 명시한다.
- **Do:** 처리하지 않는 delegate도 개별 case를 선언하고 `.none`을 반환한다. 자식별 분기 switch에도 자식 case를 각각 명시한다.
- **Don't:** delegate를 포괄하는 `case .delegate:`, `case .delegate(_)`, `default`, `case _`, `@unknown default`로 추가된 이벤트를 숨기지 않는다.
- **이유:** Delegate에 case가 추가되면 exhaustive switch의 누락이 컴파일 오류로 드러나고 부모에서 처리 여부를 결정하게 한다.
- **타입 주의:** `Action.Child`는 `.profile(ProfileFeature.Action)` 같은 래퍼다. 여기서 직접 `.delegate`를 추출하지 않는다. 자식 Action을 꺼낸 뒤 추출하며, 추출 후 switch의 타입은 Delegate이므로 `case .saved:`처럼 작성한다.
- **예외:** delegate가 아닌 자식 이벤트는 guard에서 `.none`으로 무시한다. 이것은 자식 자체의 Scope·ifLet·forEach 처리를 막지 않는다. 컬렉션 자식은 아래 예시처럼 element ID와 delegate를 함께 추출한다. Delegate 역할이 없는 자식에 빈 Delegate나 guard 처리 메서드를 만들지 않는다.
- **근거:** 사용자의 자식별 메서드 분리 및 새 Delegate case를 컴파일 오류로 발견하려는 요구

분리 메서드 이름은 아래 예시처럼 자식 case 이름을 사용해 역할별 메서드 스타일과 맞춘다. 같은 private 메서드 그룹 안에서 이름순으로 배치한다. 이름이 충돌하는 경우의 명명 정책은 아직 정하지 않았다.

## Do: 두 자식 연결과 개별 delegate 처리

ProfileFeature와 SettingsFeature의 State가 Equatable을 준수하며 기본 initializer를 제공한다고 가정한다. ProfileFeature.Action.Delegate는 `cancelled`, `saved`를, SettingsFeature.Action.Delegate는 `saved`를 가진다. 부모의 View 이벤트는 없으므로 빈 View를 추가하지 않는다.

```swift
import ComposableArchitecture

@Reducer
public struct ParentFeature {
  @ObservableState
  public struct State: Equatable {
    var profile = ProfileFeature.State()
    var saveCount = 0
    var settings = SettingsFeature.State()

    public init() {}
  }

  public enum Action {
    case child(Child)

    @CasePathable
    public enum Child {
      case profile(ProfileFeature.Action)
      case settings(SettingsFeature.Action)
    }
  }

  public init() {}

  public var body: some ReducerOf<Self> {
    Scope(\.profile, action: \.child.profile) {
      ProfileFeature()
    }
    Scope(\.settings, action: \.child.settings) {
      SettingsFeature()
    }

    Reduce { state, action in
      switch action {
      case let .child(action):
        return child(into: &state, action: action)
      }
    }
  }

  private func child(
    into state: inout State,
    action: Action.Child
  ) -> Effect<Action> {
    switch action {
    case let .profile(action):
      return profile(into: &state, action: action)

    case let .settings(action):
      return settings(into: &state, action: action)
    }
  }

  private func profile(
    into state: inout State,
    action: ProfileFeature.Action
  ) -> Effect<Action> {
    guard case let .delegate(action) = action else {
      return .none
    }

    switch action {
    case .cancelled:
      return .none

    case .saved:
      state.saveCount += 1
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
    case .saved:
      state.saveCount += 1
      return .none
    }
  }
}
```

## Don't: 새 delegate를 숨기는 포괄 분기

```swift
// ProfileFeature.Action을 받아 일부 delegate만 처리한 뒤 나머지를 숨긴다.
switch action {
case .delegate(.saved):
  state.saveCount += 1
  return .none

case .delegate:
  return .none

default:
  return .none
}
```

위 형태는 Delegate에 새 case가 추가되어도 부모에게 처리를 요구하지 않는다. guard로 delegate를 추출한 뒤 모든 case를 명시하는 형태를 사용한다.

## Optional·컬렉션 자식 연결

- **강도:** 필수
- **적용 조건:** 일반 optional 자식 또는 ID로 식별하는 컬렉션 자식을 부모에 연결할 때
- **규칙:** optional은 `ChildFeature.State?`와 `ChildFeature.Action`을 선언하고 `.ifLet`으로 연결한다. 컬렉션은 `IdentifiedArrayOf<ChildFeature.State>`와 `IdentifiedActionOf<ChildFeature>`를 선언하고 `.forEach`로 연결한다. 두 경우 모두 Action case는 `Action.Child` 내부에 둔다.
- **기본값:** optional 상태는 `nil`로 시작하되 선언부의 `= nil`은 생략하고, 컬렉션은 `[]`로 시작한다. 외부 초기값이 필요하면 기존 State 초기화 규칙에 따라 init으로 받는다.
- **컬렉션 식별:** 요소 State는 `Equatable, Identifiable`을 준수하며, public State의 `id`는 public으로 선언한다. 부모는 위치 인덱스가 아닌 이벤트에 포함된 ID로 해당 요소를 처리한다.
- **Do:** 부모 Reduce에 `.ifLet(\.editor, action: \.child.editor)` 또는 `.forEach(\.rows, action: \.child.rows)`를 붙인다. 컬렉션 처리 메서드는 `IdentifiedActionOf<RowFeature>`를 받고 element ID와 delegate를 추출한다.
- **Don't:** optional을 강제 언래핑하여 Scope에 전달하거나 컬렉션 요소마다 reducer를 직접 실행하지 않는다. 이미 nil인 자식이나 제거된 ID에 Action을 보내지 않는다.
- **이유:** ifLet과 forEach가 해당 자식 Action을 자식 reducer에서 먼저 처리한 뒤 부모를 실행하므로, 부모는 delegate에 반응하여 optional을 nil로 만들거나 요소를 제거할 수 있다.
- **예외:** 일반 optional 상태와 sheet 등의 화면 표시 상태는 구분한다. 화면 표시에는 [화면 표시](presentation.md)의 @Presents·PresentationAction 규칙을 적용한다.
- **근거:** 사용자가 승인한 ifLet·forEach 연결 및 자식별 delegate 처리 예제
- **API 근거:** [기준 커밋의 ifLet](https://github.com/pointfreeco/swift-composable-architecture/blob/377da4061db10d26337a71bb279c506bb951f50f/Sources/ComposableArchitecture/Reducer/Reducers/IfLetReducer.swift), [기준 커밋의 forEach](https://github.com/pointfreeco/swift-composable-architecture/blob/377da4061db10d26337a71bb279c506bb951f50f/Sources/ComposableArchitecture/Reducer/Reducers/ForEachReducer.swift)

### Do: State·Action과 reducer 연결

부모 Feature 내부 발췌다. EditorFeature.State는 Equatable, RowFeature.State는 Equatable·Identifiable을 준수한다고 가정한다. Editor의 Delegate는 cancelled·saved, Row의 Delegate는 deleteRequested를 가진다.

```swift
@ObservableState
public struct State: Equatable {
  var editor: EditorFeature.State?
  var rows: IdentifiedArrayOf<RowFeature.State> = []

  public init() {}
}

public enum Action {
  case child(Child)

  @CasePathable
  public enum Child {
    case editor(EditorFeature.Action)
    case rows(IdentifiedActionOf<RowFeature>)
  }
}

public var body: some ReducerOf<Self> {
  Reduce { state, action in
    switch action {
    case let .child(action):
      return child(into: &state, action: action)
    }
  }
  .ifLet(\.editor, action: \.child.editor) {
    EditorFeature()
  }
  .forEach(\.rows, action: \.child.rows) {
    RowFeature()
  }
}

private func child(
  into state: inout State,
  action: Action.Child
) -> Effect<Action> {
  switch action {
  case let .editor(action):
    return editor(into: &state, action: action)

  case let .rows(action):
    return rows(into: &state, action: action)
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
    state.editor = nil
    return .none

  case .saved:
    state.editor = nil
    return .none
  }
}

private func rows(
  into state: inout State,
  action: IdentifiedActionOf<RowFeature>
) -> Effect<Action> {
  guard case let .element(id, .delegate(action)) = action else {
    return .none
  }

  switch action {
  case .deleteRequested:
    state.rows.remove(id: id)
    return .none
  }
}
```

Delegate switch에는 default나 포괄 case를 추가하지 않는다. 새로운 delegate case가 추가되면 부모에서 처리 여부를 명시적으로 결정한다.

## Scope와 modifier를 함께 사용하는 경우

- **강도:** 조건부 필수
- **적용 조건:** 항상 존재하는 자식의 Scope와 optional·컬렉션 자식의 ifLet 또는 forEach를 함께 사용할 때
- **규칙:** Scope와 부모 Reduce를 `CombineReducers` 내부에 넣고, `.ifLet`·`.forEach`를 CombineReducers 바깥에 적용한다. 내부에서는 `Scope → Reduce` 순서를 유지한다.
- **Do:** 자식을 제거할 수 있는 부모 처리도 modifier가 감싸는 범위에 포함한다.
- **Don't:** optional·컬렉션 상태를 변경하는 reducer를 modifier 범위 밖에 두거나, Scope와 Reduce만 있는 구성에 불필요하게 CombineReducers를 추가하지 않는다.
- **이유:** modifier가 감싼 부모 처리보다 해당 자식의 처리가 먼저 실행되도록 구성한다.
- **예외:** Reduce만 있으면 직접 modifier를 붙인다. modifier 체인의 표기 순서를 전체 실행 순서로 해석하거나 알파벳순으로 기계적으로 재배열하지 않는다.
- **근거:** 사용자의 Scope와 ifLet·forEach 병용 시 CombineReducers로 묶는 결정

### Do: Scope → Reduce를 묶은 뒤 modifier 적용

위 부모에 항상 존재하는 `profile` 자식이 추가된 경우의 body 발췌다. State의 profile 프로퍼티, Action.Child의 profile case 및 child 메서드의 profile 분기가 함께 선언되어 있다고 가정한다.

```swift
public var body: some ReducerOf<Self> {
  CombineReducers {
    Scope(\.profile, action: \.child.profile) {
      ProfileFeature()
    }

    Reduce { state, action in
      switch action {
      case let .child(action):
        return child(into: &state, action: action)
      }
    }
  }
  .ifLet(\.editor, action: \.child.editor) {
    EditorFeature()
  }
  .forEach(\.rows, action: \.child.rows) {
    RowFeature()
  }
}
```
