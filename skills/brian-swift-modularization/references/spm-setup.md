# SPM 생성과 연결

패키지·타깃을 생성하거나 `Package.swift`를 수정할 때 읽는다. 배치와 재노출 범위는 [구조와 공통 의존성](conventions.md)을 따른다.

## 생성 기준

- **규칙:** 기존 계층 패키지를 재사용하고, 필요한 타깃의 소스·의존 선언·외부 제품 연결을 함께 추가한다.
- **분류:** 개인 컨벤션
- **강도:** 기본값
- **적용 조건:** SPM 패키지·타깃 생성 또는 의존 연결 변경
- **Do:** 기존 `Package.swift`를 먼저 확인하고 필요한 항목만 추가한다. 새 패키지는 `Packages/Clients`, `Packages/Features`, `Packages/Shared` 중 필요한 위치에 생성한다. 새 패키지의 도구 버전은 Swift 6.4를 사용한다. 기존 프로젝트의 명시적인 도구 버전·지원 플랫폼·외부 의존성 요구사항은 존중한다.
- **Don't:** 기존 패키지를 다시 초기화하거나, 모듈마다 새 패키지를 만들거나, 예시의 버전과 배포 대상을 기존 프로젝트에 덮어쓰지 않는다. 패키지를 생성한다는 이유로 테스트·실행 파일·빈 선택 모듈을 추가하지 않는다.
- **이유와 예외:** 패키지 관리 단위를 유지하면서 요청한 기능만 연결한다. Swift 6.4는 사용자가 정한 기본값이다. 배포 대상 등 아직 정하지 않은 선택이 구현에 영향을 주면 필요한 값을 사용자에게 확인한다.
- **근거:** 사용자의 계층별 패키지 구성, 생성 절차·manifest 예시 추가 승인 및 Swift 6.4 사용 결정
- **검토 버전:** TCA 비종속 생성 규칙. TCA 요구사항은 [TCA Feature 스킬의 현재 기준](../../brian-tca-feature/SKILL.md)을 참조한다.

## 새 패키지를 만들 때

1. 해당 계층의 `Package.swift`가 이미 있는지 확인한다. 있으면 새 패키지 대신 다음 절차로 타깃을 추가한다.
2. 필요한 계층 디렉터리에 `Package.swift`를 작성한다. 패키지명은 디렉터리와 같은 Clients·Features·Shared를 사용한다.
3. `Sources/타깃명/`을 만들고 요청한 Swift 소스를 배치한다. Core를 만드는 경우 `Exports.swift`에 합의된 재노출을 작성한다. 디렉터리만 있는 타깃을 완성된 모듈로 취급하지 않는다.
4. 외부에서 사용할 타깃의 library product와 실제 타깃 의존성을 선언한다.
5. 소비하는 패키지나 앱에 연결하고 아래 검증 절차를 수행한다.

이 구조는 manifest를 직접 작성해 생성할 수 있다. `swift package init`을 반드시 실행할 필요는 없다. 초기화 명령을 사용하는 경우 새 디렉터리에서만 실행하고, 생성된 기본 타깃과 파일을 실제 요청 범위에 맞춘다. 기존 소스나 사용 중인 설정은 지우지 않는다.

## 기존 패키지에 타깃을 추가할 때

1. `Sources/모듈명/`에 실제 소스를 추가한다.
2. `targets`에 같은 이름의 `.target`을 추가한다.
3. 사용하는 모듈을 해당 타깃의 `dependencies`에 추가한다.
4. 다른 패키지나 앱에서 사용할 때만 `products`에 `.library`를 추가한다. 같은 패키지 내부 전용 Core에는 product가 필요하지 않다.
5. 소비하는 타깃의 의존 선언과 소스의 import를 함께 수정한다. 순서에 의미가 없는 목록은 이름순으로 정렬한다.

| 연결 | 선언 위치와 형태 |
| --- | --- |
| 같은 패키지의 타깃 | 소비 타깃의 `dependencies`에 `"ClientCore"` |
| 다른 로컬 패키지 | 패키지의 `dependencies`에 `.package(path: "../Shared")` |
| 다른 패키지가 노출한 제품 | 소비 타깃의 `dependencies`에 `.product(name: "Entities", package: "shared")` |
| 외부에서 사용할 타깃 노출 | 제공 패키지의 `products`에 Entities 타깃을 노출하는 library 선언 추가 |

로컬 경로는 소비 패키지의 위치 기준이다. 아래 예시의 `package:`에는 로컬 디렉터리 이름에서 정규화된 식별자 `clients`·`shared`를 사용한다. 원격 패키지는 저장소 식별자를 사용하며 `Package(name:)`이나 import할 모듈 이름과 같다고 가정하지 않는다. 기존 의존성의 식별자가 불확실하면 SPM이 해석한 manifest·의존 그래프를 확인한다.

## 기존 코드를 모듈로 추출할 때

- **규칙:** 기존 소스의 소유 타깃을 이전하고 소비 코드·공개 API·리소스 연결을 함께 수정한다.
- **분류:** 개인 컨벤션
- **강도:** 기본값
- **적용 조건:** 앱이나 기존 모듈의 코드를 다른 SPM 타깃으로 이동할 때
- **Do:** 아래 절차로 소스 포함 설정과 사용처를 함께 확인한다. 프로젝트 생성 도구가 있으면 생성 결과 대신 원본 설정을 수정한다.
- **Don't:** 파일을 복사한 뒤 기존 타깃에서도 계속 컴파일하게 두거나, 접근 오류를 없애려고 모든 선언을 public으로 바꾸지 않는다. 리소스 소유권을 확인하지 않고 번들 접근을 일괄 변경하지 않는다.
- **이유와 예외:** 소스와 리소스의 위치가 바뀌어도 소비자의 동작을 유지한다. 의도적으로 여러 타깃이 공유하는 소스나 앱이 소유해야 하는 리소스는 기존 목적을 확인하고 필요한 연결을 유지한다.
- **근거:** 사용자의 모듈 추출 절차 보완 승인
- **검토 버전:** TCA 비종속 모듈 이동 절차

1. 이동할 소스·관련 리소스와 사용처를 확인하고, 현재 타깃의 소스 포함 방식과 빌드 설정을 파악한다.
2. 대상 `Sources/타깃명/`으로 코드를 이동하고 타깃·의존성·필요한 library product를 연결한다. 기존 타깃의 파일 참조, Compile Sources, 자동 포함 폴더 또는 생성 도구의 소스 패턴에서 이전 코드가 중복으로 포함되지 않는지 확인한다.
3. 소비 코드의 import와 필요한 모듈 한정 타입 이름을 수정한다. 모듈 경계 때문에 접근할 수 없게 된 타입·initializer·멤버 중 실제 외부 사용에 필요한 API만 공개한다. Core를 사용하는 경우 재노출 규칙을 적용한다.
4. 리소스의 소유 타깃을 결정한다. 패키지로 이동한 리소스는 필요한 SPM 리소스 선언과 패키지 번들 접근을 함께 수정하고, 앱에 남기는 리소스는 기존 앱 번들 접근을 유지하거나 필요한 값을 전달한다. 파일 위치만 옮기고 번들 접근을 그대로 두지 않는다.
5. 대상 플랫폼에서 이동한 모듈과 소비 앱을 빌드한다. 리소스가 있으면 실제 소비 경로에서 로딩도 확인한다. 확인하지 못한 실행 경로는 별도로 알린다.

## Manifest 표기

- **규칙:** 두 칸 들여쓰기를 사용하고 product·target 선언은 여러 줄로, package·외부 product 의존 선언은 한 줄로 작성한다.
- **분류:** 개인 컨벤션
- **강도:** 기본값
- **적용 조건:** `Package.swift`를 작성하거나 선언을 수정할 때
- **Do:** `.library`와 `.target`은 이름만 있어도 인자를 다음 줄에 둔다. 같은 패키지의 타깃 의존성은 문자열로 작성한다. `products`, 패키지 `dependencies`, `targets`, 타깃 `dependencies` 배열의 마지막 요소에도 쉼표를 붙인다. 짧은 `targets: ["모듈명"]` 배열은 한 줄로 유지한다.
- **Don't:** library·target 선언 전체를 한 줄로 압축하거나, 단순한 package·product 의존 선언을 여러 줄로 나누지 않는다. 함수 호출의 마지막 인자 뒤에 배열 요소용 후행 쉼표를 붙이지 않는다.
- **이유와 예외:** 선언의 구조와 의존 목록을 일관되게 읽을 수 있도록 한다. 조건부 타깃 의존처럼 문자열만으로 표현할 수 없는 설정에는 명시적 API를 사용한다. 기존 요구사항의 버전·브랜치 선택은 표기 변경과 별도로 유지한다.
- **근거:** 사용자의 manifest 표기 컨벤션 결정
- **검토 버전:** Swift 6.4. TCA 비종속 표기 규칙.

타깃 의존성은 같은 패키지의 문자열 참조를 먼저, 외부 `.product` 참조를 다음에 배치하고 각 그룹 안에서 이름순으로 정렬한다. 나머지 목록은 [알파벳 정렬](conventions.md#알파벳-정렬)을 따른다. 배열 요소 사이에 빈 줄을 넣지 않는다.

## 외부 의존성 버전 선택

- **규칙:** 기존 프로젝트의 요구사항을 우선하고, 새 TCA 요구사항의 기준이 필요할 때만 [TCA Feature 스킬의 현재 기준](../../brian-tca-feature/SKILL.md)을 참조한다.
- **분류:** 개인 컨벤션
- **강도:** 기본값
- **적용 조건:** TCA 또는 swift-dependencies의 패키지 요구사항을 새로 선언할 때
- **Do:** 기존 `Package.swift`와 `Package.resolved`를 확인한다. swift-dependencies는 기존의 명시적 요구사항을 우선하고, 없다면 선택한 TCA와 함께 사용하는 해결 버전과 호환성을 확인한다.
- **Don't:** 이 스킬에 TCA 버전 숫자를 복사하거나, swift-dependencies에 TCA 버전 숫자를 그대로 사용하거나, 모듈화만을 위해 기존 프로젝트의 버전을 올리지 않는다. 일반 작업마다 최신 릴리스를 조회하지 않는다.
- **이유와 예외:** 버전 기준은 TCA 스킬에서 관리하고 참조와 manifest 작성은 모듈화 스킬에서 처리한다. `Package.resolved`는 해결 결과이므로 기존 `from:`·`exact:` 정책과 구분한다. 필요한 기준을 읽을 수 없거나 호환성을 확인할 근거가 없으면 임의의 버전을 넣지 말고 사용자에게 확인한다.
- **근거:** 사용자의 버전 중복 관리 제거 및 스킬 간 단방향 참조 결정
- **검토 버전:** TCA 비종속 버전 선택 절차. 구체적인 TCA 기준은 위 참조를 따른다.

## 연결된 manifest 예시

아래 세 manifest는 ProductClient가 ClientCore를 통해 Entities를 사용하고, CatalogFeature가 FeatureCore와 ProductClient를 사용하는 manifest 템플릿이다. 버전 자리표시자를 아래 절차로 치환한 뒤 사용한다.

- `swift-tools-version: 6.4`는 사용자가 정한 새 패키지 기본값이다. iOS 17은 설명용 배포 대상이며 프로젝트 기준에 맞춘다. 다른 지원 플랫폼이 있으면 그 프로젝트에 맞춰 명시한다.
- 외부 버전의 `from:`은 최소 요구사항이며 실제 해석 버전을 고정하지 않는다. 기존 프로젝트의 의존성 요구사항과 해결 결과를 우선한다.
- `<TCA_VERSION>`과 `<DEPENDENCIES_VERSION>`은 실제 버전이 아니다. [외부 의존성 버전 선택](#외부-의존성-버전-선택)에 따라 값을 확인하고 실제 manifest에 넣기 전에 치환한다. 기존 요구사항이 있으면 `from:` 형식까지 덮어쓰지 말고 그대로 유지한다. 버전을 확인할 수 없으면 임의의 숫자나 최신 버전을 사용하지 말고 필요한 정보를 확인한다.
- 소스 파일은 생성 절차에 따라 별도로 작성한다. Networking·LocalDatabase는 이 예시에 필요하지 않아 포함하지 않았다.

### Packages/Shared/Package.swift

```swift
// swift-tools-version: 6.4

import PackageDescription

let package = Package(
  name: "Shared",
  platforms: [
    .iOS(.v17)
  ],
  products: [
    .library(
      name: "DesignSystem",
      targets: ["DesignSystem"]
    ),
    .library(
      name: "Entities",
      targets: ["Entities"]
    ),
  ],
  targets: [
    .target(
      name: "DesignSystem"
    ),
    .target(
      name: "Entities"
    ),
  ]
)
```

### Packages/Clients/Package.swift

```swift
// swift-tools-version: 6.4

import PackageDescription

let package = Package(
  name: "Clients",
  platforms: [
    .iOS(.v17)
  ],
  products: [
    .library(
      name: "ProductClient",
      targets: ["ProductClient"]
    ),
  ],
  dependencies: [
    .package(path: "../Shared"),
    .package(url: "https://github.com/pointfreeco/swift-dependencies", from: "<DEPENDENCIES_VERSION>"),
  ],
  targets: [
    .target(
      name: "ClientCore",
      dependencies: [
        .product(name: "Dependencies", package: "swift-dependencies"),
        .product(name: "DependenciesMacros", package: "swift-dependencies"),
        .product(name: "Entities", package: "shared"),
      ]
    ),
    .target(
      name: "ProductClient",
      dependencies: [
        "ClientCore",
      ]
    ),
  ]
)
```

### Packages/Features/Package.swift

```swift
// swift-tools-version: 6.4

import PackageDescription

let package = Package(
  name: "Features",
  platforms: [
    .iOS(.v17)
  ],
  products: [
    .library(
      name: "CatalogFeature",
      targets: ["CatalogFeature"]
    ),
  ],
  dependencies: [
    .package(path: "../Clients"),
    .package(path: "../Shared"),
    .package(url: "https://github.com/pointfreeco/swift-composable-architecture", from: "<TCA_VERSION>"),
  ],
  targets: [
    .target(
      name: "CatalogFeature",
      dependencies: [
        "FeatureCore",
        .product(name: "ProductClient", package: "clients"),
      ]
    ),
    .target(
      name: "FeatureCore",
      dependencies: [
        .product(name: "ComposableArchitecture", package: "swift-composable-architecture"),
        .product(name: "DesignSystem", package: "shared"),
        .product(name: "Entities", package: "shared"),
      ]
    ),
  ]
)
```

Core의 `Exports.swift`는 [Core와 Exports.swift](conventions.md#core와-exportsswift)의 내용을 사용한다. ProductClient 소스는 `import ClientCore`를, CatalogFeature 소스는 이름순으로 `import FeatureCore`, `import ProductClient`를 사용한다. 제품 이름은 여기서 타깃과 같지만, import 대상은 항상 모듈 이름이다.

선택 기능이 필요해지면 Shared에 해당 타깃과 library product를 추가하고, 사용하는 Client의 dependencies에만 `.product(name: "Networking", package: "shared")` 또는 LocalDatabase 제품을 추가한다. Core의 재노출 목록을 늘리지 않는다.

## 앱 연결

기존 Xcode 프로젝트·워크스페이스 또는 생성 도구의 연결 방식을 유지한다. 앱에서 사용하는 로컬 패키지를 등록하고 해당 library product를 앱 타깃의 의존·링크 대상으로 연결한다. 예를 들어 앱에서 CatalogFeature를 조립하면 Features 패키지의 CatalogFeature 제품을 연결한다. 앱 소스에서 다른 제품도 직접 사용하면 해당 제품의 연결을 확인한다.

워크스페이스에 폴더를 표시하거나 패키지 참조만 추가한 것으로 링크가 완료됐다고 판단하지 않는다. 연결 후 앱 타깃에서 모듈 import와 외부 공개 진입점 사용을 확인한다. 모듈화 작업만을 위해 프로젝트 생성 도구나 워크스페이스 구조를 교체하지 않는다.

## 검증

템플릿의 버전 자리표시자가 실제 값으로 치환됐는지 확인한 뒤 각 패키지의 manifest 평가를 우선 확인한다. 저장소 루트에서 다음처럼 실행한다.

```sh
swift package --package-path Packages/Clients dump-package
swift package --package-path Packages/Features dump-package
swift package --package-path Packages/Shared dump-package
```

`dump-package` 성공은 manifest 평가만 확인한다. 소스 컴파일, 제품 연결, 외부 버전 해결, Core의 재노출·매크로 사용을 보증하지 않는다. 이후 대상 프로젝트의 지원 플랫폼과 실제 앱 scheme에 맞춰 관련 타깃을 빌드한다. iOS 전용 코드를 macOS 호스트의 기본 `swift build`로 확인한 결과를 iOS 빌드 결과로 설명하지 않는다.

위 세 템플릿은 버전 자리표시자를 검증용 실제 값으로 치환한 뒤 Apple Swift 6.4에서 `dump-package` 평가를 확인했다. 치환 전 템플릿 자체는 실행 가능한 manifest가 아니다. 외부 의존성 해결, 소스 컴파일 및 앱 빌드는 검증하지 않았다. 실제 프로젝트에 적용한 뒤에는 그 프로젝트에서 확인한 범위를 별도로 알린다.
