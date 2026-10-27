# Alert와 ConfirmationDialog

삭제 확인·오류 안내 등의 Alert 또는 ConfirmationDialog를 연결할 때 읽는다. 기준은 TCA 1.26.2이며 아래 예제의 컴파일은 검증하지 않았다.

## 구성과 이벤트 처리

- **분류:** 개인 컨벤션
- **강도:** 기본값
- **적용 조건:** 단일 Alert 또는 ConfirmationDialog를 부모 Feature에 연결할 때
- **규칙:** State에는 `@Presents`를 사용하고, Action.Child 내부에 PresentationAction case와 버튼 Action enum을 선언한다. Child와 버튼 Action enum에는 `@CasePathable`을 적용한다. reducer는 `.ifLet`으로 연결한다.
- **Do:** child 메서드에서 alert 또는 confirmationDialog 메서드로 분기한다. 해당 메서드는 dismiss와 presented를 나누고, presented 안에서 모든 버튼 Action을 명시적으로 처리한다. 별도 작업이 없는 취소 버튼은 Action 없이 선언한다.
- **Don't:** 버튼 Action을 처리하기 위해 가짜 자식 Feature나 Delegate 계층을 만들지 않는다. presented 내부를 default나 포괄 case로 처리하지 않는다.
- **이유와 예외:** Alert·ConfirmationDialog는 자식 Feature의 delegate를 추출하는 구조와 달리 버튼 Action을 직접 받는다. 실제 작업을 부모에 요청할 때에는 현재 Feature의 Delegate를 사용하며, 현재 Feature가 담당하는 API 호출은 기존 run·Local 응답 규칙을 따른다. 취소 후 별도 작업이 필요하면 해당 버튼 Action을 명시한다.
- **닫힘:** 버튼 선택 후 닫힘과 dismiss 정리는 TCA presentation 처리에 맡긴다. 동일한 닫힘을 위해 nil 대입을 반복하지 않는다.
- **표시 대상 수:** 아래 예제는 Alert 하나 또는 ConfirmationDialog 하나를 사용하는 대안이다. 같은 부모의 상호 배타적인 표시 대상이 둘 이상이면 [Destination 구성](destination.md)에 따라 통합한다. sheet 등 다른 표시 대상도 함께 고려한다.
- **근거:** 사용자가 승인한 Child Action 구성과 버튼별 처리 예제
- **API 근거:** [기준 커밋의 Alert·ConfirmationDialog 예제](https://github.com/pointfreeco/swift-composable-architecture/blob/377da4061db10d26337a71bb279c506bb951f50f/Examples/CaseStudies/SwiftUICaseStudies/01-GettingStarted-AlertsAndConfirmationDialogs.swift)
- **검토 버전:** TCA 1.26.2

## Do: 삭제 확인 Alert

실제 삭제는 부모가 담당하는 예제다. 현재 Feature는 확인 결과를 deleteRequested로 전달한다.

```swift
import ComposableArchitecture

@Reducer
public struct ItemFeature {
  @ObservableState
  public struct State: Equatable {
    @Presents var alert: AlertState<Action.Child.Alert>?

    public init() {}
  }

  public enum Action: ViewAction {
    case child(Child)
    case delegate(Delegate)
    case view(View)

    @CasePathable
    public enum Child {
      case alert(PresentationAction<Alert>)

      @CasePathable
      public enum Alert {
        case deleteButtonTapped
      }
    }

    public enum Delegate {
      case deleteRequested
    }

    public enum View {
      case deleteButtonTapped
    }
  }

  public init() {}

  public var body: some ReducerOf<Self> {
    Reduce { state, action in
      switch action {
      case let .child(action):
        return child(into: &state, action: action)

      case let .delegate(action):
        return delegate(into: &state, action: action)

      case let .view(action):
        return view(into: &state, action: action)
      }
    }
    .ifLet(\.$alert, action: \.child.alert)
  }

  private func alert(
    into state: inout State,
    action: PresentationAction<Action.Child.Alert>
  ) -> Effect<Action> {
    switch action {
    case .dismiss:
      return .none

    case let .presented(action):
      switch action {
      case .deleteButtonTapped:
        return .send(.delegate(.deleteRequested))
      }
    }
  }

  private func child(
    into state: inout State,
    action: Action.Child
  ) -> Effect<Action> {
    switch action {
    case let .alert(action):
      return alert(into: &state, action: action)
    }
  }

  private func delegate(
    into state: inout State,
    action: Action.Delegate
  ) -> Effect<Action> {
    switch action {
    case .deleteRequested:
      return .none
    }
  }

  private func view(
    into state: inout State,
    action: Action.View
  ) -> Effect<Action> {
    switch action {
    case .deleteButtonTapped:
      state.alert = AlertState {
        TextState("항목을 삭제할까요?")
      } actions: {
        ButtonState(role: .cancel) {
          TextState("취소")
        }
        ButtonState(
          role: .destructive,
          action: .deleteButtonTapped
        ) {
          TextState("삭제")
        }
      } message: {
        TextState("삭제한 항목은 복구할 수 없어요.")
      }
      return .none
    }
  }
}
```

```swift
import ComposableArchitecture
import SwiftUI

@ViewAction(for: ItemFeature.self)
struct ItemView: View {
  @Bindable var store: StoreOf<ItemFeature>

  init(_ store: StoreOf<ItemFeature>) {
    self.store = store
  }

  var body: some View {
    Button("삭제", role: .destructive) {
      send(.deleteButtonTapped)
    }
    .alert(
      $store.scope(
        state: \.alert,
        action: \.child.alert
      )
    )
  }
}
```

## Do: ConfirmationDialog만 사용하는 경우

위 예제의 alert를 confirmationDialog로 교체한다. State는 다음과 같이 선언한다.

```swift
@Presents var confirmationDialog:
  ConfirmationDialogState<Action.Child.ConfirmationDialog>?
```

Action.Child 내부에는 다음을 선언한다.

```swift
case confirmationDialog(PresentationAction<ConfirmationDialog>)

@CasePathable
public enum ConfirmationDialog {
  case deleteButtonTapped
}
```

생성 시 `ConfirmationDialogState`를 사용하고, 처리 메서드 이름도 confirmationDialog로 변경한다. child 메서드는 해당 메서드로 전달하며, dismiss·presented 분기와 버튼별 처리는 Alert 예제와 같다.

```swift
state.confirmationDialog = ConfirmationDialogState {
  TextState("항목을 삭제할까요?")
} actions: {
  ButtonState(role: .cancel) {
    TextState("취소")
  }
  ButtonState(role: .destructive, action: .deleteButtonTapped) {
    TextState("삭제")
  }
}
```

```swift
// Reducer 연결
.ifLet(
  \.$confirmationDialog,
  action: \.child.confirmationDialog
)
```

```swift
// View 연결
.confirmationDialog(
  $store.scope(
    state: \.confirmationDialog,
    action: \.child.confirmationDialog
  )
)
```

## Don't: 버튼 Action 누락을 감추는 분기

```swift
case let .presented(action):
  switch action {
  case .deleteButtonTapped:
    return .send(.delegate(.deleteRequested))
  default:
    return .none
  }
```

버튼 Action이 추가되면 해당 처리 위치에서 컴파일 오류로 드러나도록 모든 case를 명시한다.
