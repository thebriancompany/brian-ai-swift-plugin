# 구조와 공통 의존성

아래 규칙은 사용자가 확정한 개인 컨벤션이다. Swift 또는 TCA 자체의 필수 구조로 설명하지 않는다. 검토 버전은 TCA 비종속 구조 규칙에 해당하지 않는다. Core의 코드 예제는 재노출 구성 예시이며, 컴파일은 검증하지 않았다. 실제 API 호환성은 대상 프로젝트의 Swift 및 의존성 버전으로 확인한다.

## 패키지와 타깃

- **규칙:** `Packages/Clients`, `Packages/Features`, `Packages/Shared`를 각각 하나의 로컬 SPM 패키지로 두고, 내부 모듈은 각 패키지의 타깃으로 구성한다.
- **분류:** 개인 컨벤션
- **강도:** 기본값
- **적용 조건:** 패키지·타깃을 생성하거나 기존 코드를 모듈로 추출할 때
- **Do:** 각 패키지에 `Package.swift`를 두고 소스는 `Sources/타깃명/`에 둔다. FeatureCore와 ClientCore도 해당 패키지 내부 타깃으로 둔다.
- **Don't:** 기능마다 별도 패키지를 만들거나, 이름을 PresentationLayer·DataLayer로 바꾸거나, 모든 계층을 AppModules 패키지 하나로 합치지 않는다. Interface와 Implementation을 별도 모듈로 분리하지 않는다.
- **이유와 예외:** 패키지 설정을 세 곳에 모으면서 타깃으로 기능 경계를 드러낸다. 기존 프로젝트의 명시적인 다른 구조나 사용자 요청은 존중한다. 사용하지 않는 타깃을 예시 구조만 보고 미리 생성하지 않는다.
- **근거:** 사용자의 Clients·Features·Shared 패키지 구성 승인 및 인터페이스 모듈 미분리 결정
- **검토 버전:** TCA 비종속 구조 규칙

다음은 필요한 기능이 모두 존재하는 경우의 배치 예시다. 각 소스 하위 디렉터리는 타깃이며 별도 패키지가 아니다.

```text
Packages/
├── Clients/
│   ├── Package.swift
│   └── Sources/
│       ├── ClientCore/
│       └── ProductClient/
├── Features/
│   ├── Package.swift
│   └── Sources/
│       ├── CartFeature/
│       ├── CatalogFeature/
│       └── FeatureCore/
└── Shared/
    ├── Package.swift
    └── Sources/
        ├── DesignSystem/
        ├── Entities/
        ├── LocalDatabase/
        └── Networking/
```

## Feature 경계

- **규칙:** 사용자 기능 단위로 Feature 타깃을 구성하고, 함께 이해하고 변경하는 화면은 같은 타깃에 둔다.
- **분류:** 개인 컨벤션
- **강도:** 기본값
- **적용 조건:** Feature 타깃을 추가하거나 나눌 때
- **Do:** 상품 목록과 밀접한 상품 상세는 CatalogFeature 타깃 안의 폴더로 구분한다. 별도 책임을 갖는 장바구니는 CartFeature 타깃으로 둔다.
- **Don't:** TCA Feature 타입마다 타깃을 만들거나 파일 수만으로 분리하지 않는다.
- **이유와 예외:** 한 기능을 변경할 때 필요한 맥락을 모은다. 독립적인 책임이나 실제 공유 요구가 생기면 분리한다. 특정 예시의 화면 구성을 모든 앱에 강제하지 않는다.
- **근거:** 사용자의 기능 묶음 단위 모듈 구성 승인
- **검토 버전:** TCA 비종속 구조 규칙

## 의존 방향과 공개 범위

- **규칙:** Features는 Clients와 Shared를, Clients는 Shared를 사용할 수 있으며 Shared는 Features나 Clients에 의존하지 않는다.
- **분류:** 개인 컨벤션
- **강도:** 필수
- **적용 조건:** 모듈 의존성이나 외부 공개 API를 변경할 때
- **Do:** 사용하는 타깃에 필요한 의존성을 선언하고, 다른 패키지에서 사용하는 모듈은 library product로 노출한다. 패키지 내부 타깃 연결과 외부 패키지 product 연결을 구분한다. 소비자가 실제로 사용하는 API만 공개한다.
- **Don't:** Clients가 Features를 참조하거나, Core가 자신을 사용하는 개별 Feature·Client를 참조하게 만들지 않는다. 모든 선언을 기계적으로 public으로 만들지 않는다.
- **이유와 예외:** 순환 의존을 방지하고 실제 사용하는 경계를 드러낸다. 같은 패키지 안의 타깃 간 의존도 순환하지 않게 유지한다. 기존 TCA Feature의 State·init 등 명시적인 공개 규칙은 함께 적용한다.
- **근거:** 사용자의 Shared에서 ClientCore로 역방향 의존하지 않는 구조 및 필요한 Client의 직접 의존 방식 승인
- **검토 버전:** TCA 비종속 구조 규칙

## Core와 Exports.swift

- **규칙:** 계층의 공통 기반은 Core의 `Exports.swift`에서 재노출하고, 선택 기능은 사용하는 타깃이 직접 의존하고 import한다.
- **분류:** 개인 컨벤션
- **강도:** 기본값
- **적용 조건:** Core 또는 이를 사용하는 Feature·Client의 의존성과 import를 작성할 때
- **Do:** ClientCore는 Dependencies·DependenciesMacros·Entities를, FeatureCore는 ComposableArchitecture·DesignSystem·Entities를 재노출한다. 개별 Feature와 Client는 해당 Core를 import한다. 재노출하는 모듈은 Core 타깃의 manifest 의존성에도 선언한다. 다른 스킬의 독립 예제를 적용할 때도 이 구성이 있는 프로젝트에서는 직접 import를 해당 Core import로 조정한다. 이 규칙을 적용하려고 원본 스킬의 예제를 수정하지 않는다.
- **Don't:** DependenciesClient라는 모듈 이름을 사용하지 않는다. Core를 통해 이미 제공되는 모듈을 관례적으로 다시 import하거나 모든 Client·선택 기능을 Core에 재노출하지 않는다. Exports.swift만으로 SPM 의존 선언을 대신하지 않는다.
- **이유와 예외:** 공통 기반의 반복 선언을 줄인다. Networking과 LocalDatabase는 선택 기능이므로 필요한 Client가 직접 의존한다. 기존 코드에 명시적 import가 필요한 이유가 있으면 유지한다. `@_exported`는 underscored 기능이므로 대상 툴체인에서 재노출과 매크로 사용을 확인하고 안정화된 일반 API라고 설명하지 않는다.
- **근거:** 사용자의 ClientCore 재노출 승인 및 FeatureCore에 ComposableArchitecture·Entities·DesignSystem 재노출 요청
- **검토 버전:** TCA 비종속 모듈 구성 규칙. 재노출 예제 컴파일 미검증.

```swift
// ClientCore/Exports.swift
@_exported import Dependencies
@_exported import DependenciesMacros
@_exported import Entities
```

```swift
// FeatureCore/Exports.swift
@_exported import ComposableArchitecture
@_exported import DesignSystem
@_exported import Entities
```

```swift
// ProductClient의 실제 구현 파일에서 둘 다 필요한 경우
import ClientCore
import LocalDatabase
import Networking
```

## 알파벳 정렬

- **규칙:** 순서에 실행 의미가 없는 디렉터리 표기, import, product·target·dependency 목록은 이름 기준 알파벳순으로 정렬한다.
- **분류:** 개인 컨벤션
- **강도:** 필수
- **적용 조건:** 구조를 문서화하거나 관련 import·manifest 목록을 작성·수정할 때
- **Do:** Clients → Features → Shared, CartFeature → CatalogFeature → FeatureCore 순서로 나열한다. 타깃 의존 목록은 같은 패키지의 문자열 참조, 외부 product 참조 순으로 그룹을 나누고 각 그룹에서 모듈·제품 이름순으로 정렬한다. 그 밖의 의존 목록은 패키지 이름·식별자순으로 정렬한다.
- **Don't:** 추가한 시점이나 역할별 선호 순서로 뒤에 누적하지 않는다. 실행 순서에 의미가 있는 reducer·마이그레이션·처리 로직을 알파벳순으로 바꾸지 않는다.
- **이유와 예외:** 사람과 AI가 같은 기준으로 위치를 예측하게 한다. manifest API의 인자 순서나 의미 있는 실행 순서는 그대로 유지한다.
- **근거:** 사용자의 패키지·모듈 구성 알파벳순 정렬 요청
- **검토 버전:** TCA 비종속 구조 규칙
