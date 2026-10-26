# 화면 표시

sheet 같은 화면 표시를 기존 Action.Child 구조에 연결할 때 읽는다. 기준은 TCA 1.26.2다. 예제는 EditorFeature와 EditorView가 별도로 구현되어 있다고 가정한 발췌이며 전체 컴파일은 검증하지 않았다.

## 개별 @Presents와 Destination 선택

- **강도:** 조건부 필수
- **적용 조건:** 표시 상태를 처음 설계하거나 표시 대상을 추가할 때
- **규칙:** 표시 대상이 하나면 개별 `@Presents`를 사용한다. 같은 부모가 관리하는 상호 배타적인 표시 대상이 두 개 이상이면 [Destination 구성](destination.md)에 따라 통합한다. 처음부터 여러 표시 대상이 요구사항으로 확정되어 있다면 Destination으로 시작한다.
- **Do:** 편집 sheet 하나만 있으면 `editor`로 관리한다. 같은 부모에서 편집 또는 설정 중 하나를 표시하게 되면 Destination의 editor·settings case로 전환한다.
- **Don't:** 미래에 화면이 늘어날 가능성만으로 단일 표시 대상을 Destination으로 감싸지 않는다. 저장소 전체나 서로 다른 Feature의 `@Presents` 개수만 세어 일괄 통합하지 않는다.
- **소유권:** 편집 sheet 안에서 추가 화면을 표시하는 등 중첩 화면은 해당 화면을 소유한 자식 Feature가 관리한다. 부모의 Destination은 편집 화면을 유지하고, 편집 Feature가 자신의 표시 상태를 가진다.
- **이유:** 현재 필요한 구조만 도입하면서, 같은 부모의 상호 배타적인 화면은 하나의 상태로 표현한다.
- **예외:** 독립적으로 관리해야 하는 표시 상태를 개수만으로 같은 Destination에 묶지 않는다. 판단 기준은 동일한 소유자와 상호 배타성이다.
- **근거:** 사용자가 승인한 단일 표시 대상의 개별 @Presents 사용, 복수 표시 대상의 조건부 Destination 전환 및 중첩 화면 소유권 기준

Destination으로 전환할 때도 아래 dismiss/presented 처리 원칙은 유지한다.

## State·Action과 reducer 연결

- **강도:** 필수
- **적용 조건:** 자식 Feature를 sheet로 표시할 때
- **규칙:** 부모 State에 `@Presents var editor: EditorFeature.State?`을 선언하고, Action.Child에는 `case editor(PresentationAction<EditorFeature.Action>)`를 선언한다. reducer에서는 `.ifLet(\.$editor, action: \.child.editor)`로 연결한다.
- **Do:** 화면을 열 때 자식 State를 생성한다. 자식의 저장·취소 delegate에 반응하여 닫는 경우 부모에서 해당 State를 nil로 만든다.
- **Don't:** presentation을 일반 optional 연결과 혼동하여 reducer에 `\.editor`를 전달하거나, 자식 Action의 PresentationAction 래퍼를 생략하지 않는다.
- **이유:** 표시 상태와 자식 이벤트뿐 아니라 dismiss 이벤트까지 TCA의 presentation 생명주기로 관리한다.
- **예외:** Scope도 함께 있으면 [자식 Feature 연결](child-features.md)의 CombineReducers 규칙을 적용한다. Scope → Reduce를 묶고 바깥에 presentation용 ifLet을 붙인다. 일반 optional 및 컬렉션 연결은 기존 규칙을 유지한다.
- **근거:** 사용자가 승인한 @Presents·PresentationAction·ifLet 연결 방식
- **API 근거:** [기준 커밋의 PresentationReducer](https://github.com/pointfreeco/swift-composable-architecture/blob/377da4061db10d26337a71bb279c506bb951f50f/Sources/ComposableArchitecture/Reducer/Reducers/PresentationReducer.swift)

## PresentationAction 처리

- **강도:** 필수
- **적용 조건:** 부모의 자식별 메서드에서 자식 Feature의 PresentationAction을 처리할 때
- **규칙:** 먼저 `.dismiss`와 `.presented(action)`을 명시적으로 분기한다. `.presented` 내부에서 guard로 delegate를 추출한 뒤 Delegate의 모든 case를 명시한다.
- **Do:** `.dismiss`에서 추가 처리가 없으면 `.none`을 반환한다. `.presented`의 delegate switch는 처리하지 않는 case도 명시하고 `.none`을 반환한다.
- **Don't:** PresentationAction 전체에 delegate 추출 guard를 적용해 dismiss 분기를 숨기거나, Delegate switch에 default·포괄 case를 추가하지 않는다.
- **이유:** 화면 닫기 이벤트와 자식 내부 이벤트를 구분하면서 새로운 Delegate case의 처리 누락을 컴파일 시 발견한다.
- **생명주기:** `.presented`는 자식 reducer 처리 후 부모가 처리한다. `.dismiss`는 부모가 처리한 뒤 TCA가 해당 표시 상태를 정리하므로 단순 닫기를 위한 중복 nil 대입은 하지 않는다. 부모가 닫기 전에 자식 상태를 확인할 필요가 있다면 dismiss 분기에서 처리한다.
- **예외:** Delegate가 없는 자식에 빈 Delegate를 만들지 않는다. 이 경우 presented에서 필요한 부모 처리가 없으면 `.none`을 반환한다. Destination은 [화면별 분기](destination.md)를 거친다. Alert·ConfirmationDialog는 [버튼 Action 처리](alerts.md)를 적용하며 delegate 추출을 요구하지 않는다.
- **근거:** 사용자의 dismiss/presented 분리 및 presented 내부 delegate 추출 결정

## Do: 부모 Feature

EditorFeature.State는 Equatable을 준수하고 기본 initializer를 제공한다. EditorFeature.Action.Delegate에는 cancelled와 saved가 있다고 가정한다. 자식 타입의 구현은 생략했다. 아래는 부모가 닫기를 결정하는 사례이며 cancelled·saved의 사용을 모든 화면에 강제하지 않는다. 직접 닫기와 상위 처리의 선택은 아래 화면 닫기 규칙을 따른다.

```swift
import ComposableArchitecture

@Reducer
public struct ParentFeature {
  @ObservableState
  public struct State: Equatable {
    @Presents var editor: EditorFeature.State?

    public init() {}
  }

  public enum Action: ViewAction {
    case child(Child)
    case view(View)

    @CasePathable
    public enum Child {
      case editor(PresentationAction<EditorFeature.Action>)
    }

    public enum View {
      case editButtonTapped
    }
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
    .ifLet(\.$editor, action: \.child.editor) {
      EditorFeature()
    }
  }

  private func child(
    into state: inout State,
    action: Action.Child
  ) -> Effect<Action> {
    switch action {
    case let .editor(action):
      return editor(into: &state, action: action)
    }
  }

  private func editor(
    into state: inout State,
    action: PresentationAction<EditorFeature.Action>
  ) -> Effect<Action> {
    switch action {
    case .dismiss:
      return .none

    case let .presented(action):
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
  }

  private func view(
    into state: inout State,
    action: Action.View
  ) -> Effect<Action> {
    switch action {
    case .editButtonTapped:
      state.editor = EditorFeature.State()
      return .none
    }
  }
}
```

## Do: ViewAction과 sheet 바인딩

View에는 `@ViewAction(for: ParentFeature.self)`를 적용한다. UI 이벤트는 `send(.editButtonTapped)`로 전달한다. sheet 클로저의 자식 Store는 `store`로 이름 짓고 `EditorView(store: store)`에 전달한다.

```swift
import ComposableArchitecture
import SwiftUI

@ViewAction(for: ParentFeature.self)
struct ParentView: View {
  @Bindable var store: StoreOf<ParentFeature>

  var body: some View {
    Button("편집") {
      send(.editButtonTapped)
    }
    .sheet(
      item: $store.scope(
        state: \.editor,
        action: \.child.editor
      )
    ) { store in
      EditorView(store: store)
    }
  }
}
```

reducer의 ifLet은 `\.$editor`를 사용하고 View의 scope는 `\.editor`를 사용한다. View와 Feature가 다른 모듈이라면 실제 접근에 필요한 프로퍼티만 public으로 공개한다.

## Don't: dismiss를 숨기고 UI 이벤트를 직접 포장

```swift
// PresentationAction 전체에서 추출하면 dismiss 처리 위치가 사라진다.
guard case let .presented(.delegate(action)) = action else {
  return .none
}

// @ViewAction View의 UI 이벤트는 send를 사용한다.
store.send(.view(.editButtonTapped))
```

## 화면 닫기와 상위 처리

- **분류:** 개인 컨벤션
- **강도:** 기본값
- **적용 조건:** 자식 Feature에서 화면을 닫거나 상위 처리를 요청할 때
- **규칙:** 비즈니스 요구사항에 따라 결정한다. 자식에서 직접 닫을 수 있으면 `@Dependency(\.dismiss)`로 self-dismiss하고, 상위 처리가 필요하면 Delegate로 전달한다.
- **Do:** 자식의 dismiss는 run 안에서 호출한다. 상위에 전달할 결과나 판단할 일이 있으면 의미 있는 Delegate 이벤트를 선언하고, 부모가 필요한 처리와 화면 전환을 결정한다.
- **Don't:** 취소는 항상 self-dismiss, 저장은 항상 Delegate처럼 이벤트 이름만으로 방식을 고정하지 않는다. 닫기만을 위한 Delegate를 기계적으로 만들거나 동일한 닫힘을 자식과 부모에서 중복 수행하지 않는다.
- **이유와 예외:** 화면별 업무 책임을 기준으로 결정한다. 기존 예제의 cancelled·saved Delegate는 상위에서 처리하는 사례이며 모든 화면에 필수인 이벤트가 아니다. 기존 Delegate를 이름만 보고 일괄 제거하지 않는다.
- **API 조건:** dismiss는 TCA의 presentation용 ifLet 또는 스택용 forEach로 연결된 화면에서 사용한다. 일반 optional·컬렉션 연결이나 루트 Feature에서 항상 사용 가능하다고 가정하지 않는다.
- **근거:** 사용자의 비즈니스별 판단, 가능한 경우 self-dismiss 및 상위 처리 필요 시 Delegate 선택
- **API 근거:** [기준 커밋의 DismissEffect](https://github.com/pointfreeco/swift-composable-architecture/blob/377da4061db10d26337a71bb279c506bb951f50f/Sources/ComposableArchitecture/Dependencies/Dismiss.swift)
- **검토 버전:** TCA 1.26.2. 아래 발췌의 컴파일은 검증하지 않았다.

```swift
@Dependency(\.dismiss) private var dismiss
```

```swift
// view 메서드에서 직접 닫을 수 있는 경우
case .closeButtonTapped:
  return .run { [dismiss] _ in
    await dismiss()
  }
```

```swift
// 상위에서 저장 결과를 처리해야 하는 경우의 local 메서드 발췌
case let .saveResponse(.success(item)):
  state.isSaving = false
  return .send(.delegate(.saved(item)))
```

위 발췌의 Action·State·결과 타입은 해당 Feature에 선언되어 있다고 가정한다. 부모가 Delegate를 받아 닫기로 결정하면 개별 presentation State 또는 destination을 nil로 만들거나, 스택의 해당 요소부터 pop한다. 닫는 시점도 업무 요구사항에 따른다.
