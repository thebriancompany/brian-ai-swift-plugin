# Feature 컨벤션

아래 규칙은 사용자가 확정한 개인 컨벤션이다. TCA 자체의 필수 요구사항으로 설명하지 않는다. 사용자 요청과 대상 프로젝트의 명시적 규칙이 충돌하면 해당 맥락을 우선한다.

이 문서의 예제는 스킬 기준인 TCA 1.26.2를 대상으로 한 선언 구조 설명용 발췌다. 사용자가 제공한 형태를 바탕으로 작성했으며, 완성된 Feature나 컴파일 검증된 예제가 아니다. 생략된 reducer 구현 등을 완성된 코드로 간주하지 않는다.

## Feature 내부 구성

- **강도:** 필수
- **적용 조건:** Feature를 생성하거나 선언 구조를 수정할 때
- **규칙:** State, Action, reducer 구현을 동일한 Feature 내부에 함께 둔다.
- **Do:** `SomeFeature` 안에 State와 Action을 중첩 선언하고 reducer 구현도 함께 둔다.
- **Don't:** State와 Action을 각각 `SomeFeature+State.swift`, `SomeFeature+Action.swift`로 분리하거나 Feature 밖의 별도 타입으로 꺼내지 않는다.
- **이유:** Feature의 상태, 이벤트, 처리 로직을 함께 확인할 수 있는 구성을 사용자가 선택했다.
- **예외:** 크기에 따른 자동 파일 분리 예외는 정하지 않았다. 임의로 만들지 않는다.
- **화면 타입 배치:** 필요한 타입만 `State → Action → Destination → Path → CancelID → Dependency → init → body → 처리 메서드` 순으로 배치한다. Destination과 Path는 부모 Feature 내부에 선언하고, 생성된 State의 Equatable extension은 같은 파일 하단에 둔다. 세부 구성은 [Destination 구성](destination.md)과 [스택 화면 전환](stack-navigation.md)을 따른다.
- **근거:** 사용자의 Feature 내부 배치 및 Destination·Path 위치 결정

## State 기본 선언

- **강도:** 기본값
- **적용 조건:** Feature의 State를 작성하거나 선언을 수정할 때
- **규칙:** State에 `@ObservableState`를 붙이고 `Equatable`을 준수한다.
- **Do:** `@ObservableState`와 `struct State: Equatable`을 함께 사용한다.
- **Don't:** 별도 이유 없이 매크로 또는 `Equatable` 준수를 생략하지 않는다.
- **이유:** 관찰 가능한 상태와 동등성 비교를 State의 기본 형태로 통일한다.
- **예외:** 구체적인 예외는 아직 정하지 않았다. 적용에 문제가 있으면 원인을 설명하고 예외를 논의한다.
- **근거:** 사용자의 State 기본 선언 결정

## State 기본값과 초기화

- **강도:** 기본값
- **적용 조건:** State 프로퍼티와 initializer를 작성하거나 수정할 때
- **규칙:** 프로퍼티 선언부에 기본값을 둔다. nil로 시작하는 옵셔널 var는 아래 규칙에 따라 `= nil`을 생략한다. 외부에서 초기값을 전달할 필요가 있는 프로퍼티만 `init` 파라미터로 노출한다.
- **Do:** 내부 로딩 상태는 `var isLoading = false`로 선언하고, 외부에서 필요한 `id`, `name`만 initializer로 받는다.
- **Don't:** 모든 프로퍼티를 기계적으로 initializer 파라미터에 추가하거나 선언부의 기본값을 initializer에서 불필요하게 재대입하지 않는다.
- **이유:** 내부 기본 상태는 선언부에서 확인하고, 외부에서 구성해야 하는 값만 초기화 인터페이스에 드러낸다.
- **예외:** 외부에서 반드시 받아야 하는 값에는 임의의 ID나 빈 문자열 같은 가짜 기본값을 만들지 않는다. 필요한 입력으로 받는다.
- **근거:** 사용자의 선언부 기본값 및 필요한 외부 입력만 initializer에 노출하는 결정

## 옵셔널 프로퍼티의 초기값

- **분류:** 개인 컨벤션
- **강도:** 기본값
- **적용 조건:** nil로 시작하는 옵셔널 var 프로퍼티를 선언할 때
- **규칙:** 명시적인 `= nil`을 생략한다.
- **Do:** `var errorMessage: String?`, `@Presents var editor: EditorFeature.State?`
- **Don't:** 선언부에 불필요한 nil 초기값을 추가하지 않는다.
- **이유와 예외:** nil 기본값을 간결하게 표현한다. 실행 중 상태를 초기화하는 `state.errorMessage = nil`은 유지하며, 별도 초기화를 요구하는 프로퍼티 래퍼 등에는 해당 API의 초기화 요구사항을 따른다.
- **근거:** 사용자의 옵셔널 선언 시 nil 초기값 생략 결정
- **검토 버전:** TCA 1.26.2. 예제 컴파일은 검증하지 않았다.

## State 접근 제어

- **강도:** 필수
- **적용 조건:** State와 initializer, 프로퍼티를 선언하거나 접근 범위를 변경할 때
- **규칙:** State와 initializer는 `public`으로 선언한다. 프로퍼티는 외부 접근이 필요한 경우에만 `public`으로 선언한다.
- **Do:** `public State`가 `Identifiable`을 준수해야 하는 경우 요구 프로퍼티를 `public let id: UUID`처럼 노출한다. 내부 구현용 프로퍼티는 필요한 접근 범위를 사용한다.
- **Don't:** State가 public이라는 이유로 모든 프로퍼티를 public으로 만들지 않는다. `init`으로 값을 받는다는 이유만으로 해당 프로퍼티까지 public으로 만들지 않는다.
- **이유:** 외부에 필요한 인터페이스만 공개한다.
- **예외:** 모든 State에 `Identifiable`을 추가하는 규칙은 아니다. 필요한 경우에만 준수한다.
- **근거:** 사용자의 State·init 공개 범위 및 Identifiable 예시

## 접근 제어와 알파벳 정렬

- **강도:** 필수
- **적용 조건:** import, State 프로퍼티·초기화 구문, Action 선언, 역할별 처리 메서드를 추가하거나 수정할 때
- **규칙:** 접근 제어가 있는 선언은 같은 종류의 선언 그룹 안에서 `public → internal → fileprivate → private` 순으로 배치하고, 각 접근 제어 그룹 안에서는 이름의 알파벳 순서를 따른다. 접근 제어자를 생략한 프로퍼티는 internal 그룹으로 취급한다. 구체적인 적용 단위는 다음과 같다.
  - import: 모듈 이름
  - State 프로퍼티: 접근 제어 순서 우선, 같은 접근 제어 안에서는 프로퍼티 이름순. 저장·계산 프로퍼티를 함께 정렬하며 `let`과 `var`로도 그룹을 나누지 않는다.
  - State initializer: 파라미터 이름 및 대응하는 `self` 대입 순서
  - Action: 역할 case, 역할별 중첩 enum, 각 중첩 enum의 이벤트 case 이름
  - 역할별 처리 메서드: 같은 접근 제어 그룹 안에서 `child`, `delegate`, `external`, `local`, `view` 순서. 존재하는 역할만 배치한다.
- **Do:** public `id`, internal `isEnabled`·`isLoading`·`name`·`password`, private `isEmpty` 순으로 프로퍼티를 배치한다. initializer의 파라미터와 대입문은 별도로 이름순으로 정렬한다.
- **Don't:** 접근 제어를 무시한 전체 이름순 정렬, 저장·계산 프로퍼티별 분리, 변경 가능 여부나 추가 시점에 따른 정렬을 하지 않는다.
- **이유:** 선언 위치를 예측할 수 있도록 사용자가 선택한 정렬 기준을 일관되게 적용한다.
- **예외:** 실행 순서에 의미가 있는 reducer 조합, Effect 및 일반 로직을 이 규칙으로 재배치하지 않는다. Feature의 State·Action·body 같은 서로 다른 종류의 구성 요소 전체를 재배치하는 규칙도 아니다. `public init`은 접근 제어와 무관하게 State의 프로퍼티 선언 뒤에 둔다.
- **근거:** 사용자의 import·State·Action 알파벳 정렬 결정, 접근 제어 우선순위 및 State 초기화 예시

### Do: import와 State

```swift
import ComposableArchitecture
import Foundation

@Reducer
public struct SomeFeature {
  @ObservableState
  public struct State: Equatable, Identifiable {
    public let id: UUID

    var isEnabled: Bool {
      !isLoading && !isEmpty
    }
    var isLoading = false
    var name: String
    var password = ""

    private var isEmpty: Bool {
      name.isEmpty || password.isEmpty
    }

    public init(
      id: UUID,
      name: String
    ) {
      self.id = id
      self.name = name
    }
  }
}
```

위 코드는 사용자가 제시한 Identifiable State 예시를 반영한다. `id`는 public이며, `name`은 외부에서 초기값을 받지만 프로퍼티 접근은 internal이다. `isEnabled`는 같은 internal 그룹에서 이름순으로 배치하고, private `isEmpty`는 internal 프로퍼티 뒤에 둔다. initializer는 모든 프로퍼티 뒤에 위치한다.

### Don't: 선언 종류에 따른 그룹화

```swift
import Foundation
import ComposableArchitecture

// State 내부 발췌: let을 먼저 모으면 이름 순서가 깨진다.
let id: UUID
let name: String
var isLoading: Bool
```

### Don't: 접근 제어보다 이름순 정렬을 우선

```swift
// State 내부 발췌: private 프로퍼티를 internal보다 먼저 두지 않는다.
public let id: UUID
private var isEmpty: Bool { name.isEmpty }
var isLoading = false
var name: String
```

## Action 역할 분리

- **강도:** 필수
- **적용 조건:** Feature의 Action을 작성하거나 수정할 때
- **규칙:** 역할별 중첩 enum과 이를 받는 Action case로 이벤트를 구분한다. View 이벤트가 있는 Action은 사용자 예시처럼 `ViewAction`을 준수한다.
- **Do:** 필요한 역할만 아래 순서에 맞춰 선언한다. Child 중첩 enum에는 `@CasePathable`을 적용한다.
- **Don't:** UI 입력, 내부 처리, 부모 전달 이벤트를 하나의 평평한 Action에 섞지 않는다. 모든 Feature에 다섯 역할을 의무적으로 만들지 않는다.
- **이유:** 이벤트의 출처와 전달 방향을 선언 구조로 구분한다.
- **예외:** View 이벤트가 없는 Feature의 `ViewAction` 준수 방식은 아직 정하지 않았다. 준수를 위해 빈 View enum을 만들지 않는다.
- **근거:** 사용자의 역할 정의, Action 구조 및 불필요한 역할 생략 결정

| 순서 | Action case / 중첩 enum | 의미 |
| --- | --- | --- |
| 1 | `child(Child)` / `Child` | 자식 Feature의 Action 및 Alert·ConfirmationDialog의 버튼 이벤트 |
| 2 | `delegate(Delegate)` / `Delegate` | 현재 Feature에서 상위로 올리는 이벤트 |
| 3 | `external(External)` / `External` | 외부에서 현재 Feature로 보내는 이벤트 |
| 4 | `local(Local)` / `Local` | 현재 Feature 내부에서 내부로 보내는 이벤트 |
| 5 | `view(View)` / `View` | UI에서 현재 Feature로 보내는 이벤트 |

역할이 필요 없으면 해당 Action case와 중첩 enum을 모두 생략한다. 미래에 사용할 가능성만으로 빈 enum, placeholder case 또는 의미 없는 이벤트를 추가하지 않는다.

### Do: Local과 View만 필요한 경우

SomeFeature 내부의 Action 선언 발췌다. 구체적인 이벤트 이름은 역할 분류 설명용이며 접미사 등의 명명 규칙을 확정하는 예제가 아니다.

```swift
public enum Action: ViewAction {
  case local(Local)
  case view(View)

  public enum Local {
    case loadingFinished
  }

  public enum View {
    case refreshButtonTapped
    case task
  }
}
```

### Don't: 사용하지 않는 Delegate 선언

상위에 전달할 이벤트가 없는 Feature에 아래 선언을 추가하지 않는다.

```swift
// Action 내부 발췌
case delegate(Delegate)

public enum Delegate {}
```

## 배열·목록 요소 사이의 빈 줄

- **강도:** 필수
- **적용 조건:** 여러 줄 배열 또는 `.merge`처럼 여러 항목을 나열하는 인자 목록을 작성할 때
- **규칙:** 요소 사이에 빈 줄을 넣지 않는다. 쉼표 뒤 다음 줄에 바로 다음 요소를 작성한다. 요소 자체가 여러 줄인 경우 그 구조는 유지한다.
- **Do:** `.cancellable(...),` 다음 줄에 바로 다음 `.run`을 작성한다.
- **Don't:** 요소를 구분하려고 쉼표 다음에 빈 줄을 추가하지 않는다. 이 규칙을 목록 전체를 한 줄로 압축하라는 의미로 해석하지 않는다.
- **이유:** 사용자가 지정한 연속된 목록 표기를 유지한다.
- **예외:** 요소 내부 로직의 의미 있는 빈 줄이나 switch case 사이의 기존 간격까지 제거하는 규칙은 아니다.
- **근거:** 사용자의 merge 예시 및 배열 관련 요소 간 빈 줄 제거 요청

```swift
// Do
let values = [
  first,
  second
]

// Don't
let values = [
  first,

  second
]
```

## Dependency 선언

Client 자체의 선언·등록을 작성할 때는 [Dependency Client](dependency-clients.md)를 읽는다.

- **강도:** 필수
- **적용 조건:** Feature에서 사용하는 Dependency를 선언하거나 수정할 때
- **규칙:** State·Action 및 필요한 Destination·Path·CancelID 선언 뒤, Feature의 `init`·`body` 앞에 Dependency를 모아 선언한다. 각 프로퍼티는 `private`으로 두고 `@Dependency`와 같은 줄에 작성한다.
- **클라이언트 선언:** UserClient 같은 타입 기반 의존성은 `@Dependency(UserClient.self) private var userClient` 형태를 사용한다. 기존 clock·dismiss·uuid 등 키 경로 기반 의존성은 해당 API 형식을 유지한다. 타입 기반 접근을 지원하지 않는 의존성을 이 스타일만을 위해 임의로 재구성하지 않는다.
- **정렬:** 의존성 타입이나 키 경로가 아닌 선언한 프로퍼티 이름 기준으로 알파벳순 정렬한다. 예를 들어 `@Dependency(\.continuousClock) private var clock`은 `clock` 기준이다.
- **Do:** 실제 사용하는 의존성만 선언하고 Dependency 그룹 내부에서 이름순으로 정렬한다.
- **Don't:** 미사용 의존성을 미리 추가하거나, 접근 제어 전체 정렬을 이유로 Dependency를 public `body` 뒤로 옮기지 않는다. 선언을 여러 위치에 흩어 놓거나 매크로와 프로퍼티를 두 줄로 나누지 않는다.
- **이유:** Feature의 외부 의존성을 한곳에서 확인하고 선언 형태와 순서를 통일한다.
- **예외:** 접근 제어 정렬은 같은 선언 그룹 내부에 적용한다. Dependency 그룹과 `init`, `body`, 역할별 처리 메서드는 각자의 배치 순서를 유지한다.
- **근거:** 사용자가 승인한 Dependency 위치, private 접근 제어, 프로퍼티 이름순 정렬, 한 줄 선언 및 실제 사용하는 의존성만 추가하는 제안

### Do: 선언 위치와 형식

Feature 내부의 배치 발췌다. 아래 의존성은 실제 처리 코드에서 사용한다는 전제로 나열했으며, 모든 Feature에 복사할 기본 목록이 아니다.

```swift
// State, Action 선언 다음

@Dependency(\.continuousClock) private var clock
@Dependency(\.dismiss) private var dismiss
@Dependency(UserClient.self) private var userClient
@Dependency(\.uuid) private var uuid

public init() {}

// public var body: some ReducerOf<Self> { ... }
// private 역할별 처리 메서드
```

### Don't: 두 줄 선언과 잘못된 정렬

```swift
@Dependency(\.uuid)
private var uuid

@Dependency(\.continuousClock)
private var clock
```

## Reducer 역할별 처리

- **강도:** 필수
- **적용 조건:** Feature의 reducer에서 Action을 처리할 때
- **규칙:** `body`의 `Reduce`는 Action 역할을 구분하고, 실제 처리는 같은 Feature 내부의 역할별 메서드로 전달한다. 메서드 이름은 Action case와 동일한 `child`, `delegate`, `external`, `local`, `view`를 사용한다.
- **Do:** `.local(action)`은 `local(into: &state, action: action)`로, `.view(action)`은 `view(into: &state, action: action)`로 전달한다. 각 메서드는 해당 역할의 Action 타입을 받아 `Effect<Action>`을 반환한다.
- **Don't:** `reduceLocal`, `handleView` 같은 별도 접두사를 붙이거나, 역할별 실제 처리 로직을 body의 하나의 switch 안에 모두 작성하지 않는다.
- **배치:** 처리 메서드는 같은 Feature 내부에 private으로 두고 이름순으로 정렬한다. 사용하지 않는 역할의 메서드는 만들지 않는다.
- **이유:** Action의 역할 구분을 처리 코드에도 유지하고 역할명과 메서드명을 일치시킨다.
- **예외:** 이 분리는 역할별 이벤트 처리 방식에 관한 규칙이다. 자식 reducer를 연결하는 Scope, ifLet, forEach 등의 구성이나 실행 순서를 대체하는 규칙이 아니다. Scope, ifLet, forEach의 연결과 CombineReducers 구성은 [자식 Feature 연결](child-features.md)을 따른다.
- **근거:** 사용자가 선택한 역할별 메서드 분리 방식과 역할명 그대로의 메서드 명명 결정

### Do: Local과 View 처리 분리

Feature 내부의 처리 구조 발췌다. 위 Action 예제의 이벤트를 사용하며, 비동기 요청 구현은 포함하지 않는다. 실제 `.task` 로딩은 [Effect 처리](effects.md)의 요청·finish 예제를 따른다.

```swift
public var body: some ReducerOf<Self> {
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
  case .loadingFinished:
    state.isLoading = false
    return .none
  }
}

private func view(
  into state: inout State,
  action: Action.View
) -> Effect<Action> {
  switch action {
  case .refreshButtonTapped:
    state.isLoading = true
    return .none

  case .task:
    return .none
  }
}
```

### Don't: 역할과 다른 메서드 이름

```swift
// body 내부 발췌
case let .local(action):
  return reduceLocal(into: &state, action: action)

case let .view(action):
  return handleView(into: &state, action: action)
```

## 자식 Feature 연결

자식 State·Action을 추가하거나 자식 이벤트 처리를 수정할 때 [자식 Feature 연결](child-features.md)을 읽는다. Scope·ifLet·forEach 선택, CombineReducers 구성 및 delegate 처리의 상세 규칙은 해당 문서를 따른다.

## View의 UI 이벤트 전달

- **강도:** 필수
- **적용 조건:** Feature의 View Action을 보내는 SwiftUI View를 작성하거나 수정할 때
- **규칙:** View에 `@ViewAction(for: Feature.self)`를 적용하고 버튼 등 명시적 UI 이벤트는 `send(.이벤트)`로 보낸다. TextField·Toggle의 양방향 바인딩은 [바인딩](bindings.md)에 따라 `$store.프로퍼티`로 연결한다. Feature.Action은 ViewAction을 준수하고 View 이벤트를 중첩 View enum에 둔다.
- **Do:** `@ViewAction(for: ParentFeature.self)`와 `send(.editButtonTapped)`를 사용한다. sheet 바인딩이 필요한 Store는 `@Bindable`로 선언한다.
- **Don't:** View의 UI 이벤트를 `store.send(.view(.editButtonTapped))`로 직접 감싸서 보내지 않는다.
- **이유:** UI 이벤트를 View Action 경로로 일관되게 전달한다.
- **예외:** View 이벤트가 없는 화면에 빈 View Action을 만들기 위한 규칙이 아니다.
- **근거:** 사용자의 ViewAction 매크로와 send 사용 결정

sheet의 State·Action·reducer 및 View 연결은 [화면 표시](presentation.md)를 따른다. sheet 클로저의 자식 Store 이름은 `store`로 사용한다.

## 양방향 바인딩

TextField·Toggle 등 상태 바인딩을 추가하거나 수정할 때 [바인딩](bindings.md)을 읽는다. Action.View 선언, BindingReducer 순서, `$store.프로퍼티` 연결 및 변경 후 추가 작업의 규칙은 해당 문서를 따른다.

## 비동기 작업과 Effect

API 호출·응답 처리, 취소·debounce, `.task` 로딩과 `.refreshable` 새로고침을 구현할 때 [Effect 처리](effects.md)를 읽는다.

## 아직 정하지 않은 항목

- initializer 파라미터의 기본 인자 및 파생 상태의 추가 설계 기준
- View 이벤트가 없는 Feature의 Action 프로토콜 준수 방식
- 이미 확정한 `.task`·`.refresh` 등을 제외한 세부 이벤트 명명 규칙

미정 항목은 대상 프로젝트의 명시적 지침과 인접 구현을 따른다. 예제의 우연한 형태를 새로운 개인 필수 규칙으로 확대하지 않는다.
