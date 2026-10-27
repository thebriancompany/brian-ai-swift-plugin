---
name: brian-tca-feature
description: Brian의 개인 컨벤션에 맞춰 TCA Feature를 작성하거나 수정한다. 테스트 작성과 일반 Swift 작업은 범위에 포함하지 않는다.
---

# Brian TCA Feature

현재 기준: **TCA 1.26.2**

## 적용 범위

- [개인 컨벤션](references/conventions.md)을 적용하여 Feature를 작성·수정한다.
- 사용자 요청과 대상 프로젝트의 명시적인 규칙을 존중한다.
- 테스트 작성은 이 스킬의 범위에 포함하지 않는다.

## Feature 작성·수정

1. 대상 Feature, 인접 구현, 프로젝트 지침을 확인한다.
2. [개인 컨벤션](references/conventions.md)의 선언 구성, 정렬, Action 역할과 처리 메서드 규칙을 적용한다. Feature View는 `init(_ store: StoreOf<Feature>)`로 Store를 외부에서 주입받으며, 공개 View는 이 initializer도 `public`으로 선언한다. 아래 표에서 작업에 해당하는 참고 자료만 읽는다.
3. 변경한 코드와 관련된 빌드·진단을 가능한 범위에서 확인한다. 확인하지 못한 부분을 검증 완료로 표현하지 않는다.

| 작업 | 참고 자료 |
| --- | --- |
| 일반·optional·컬렉션 자식 연결, Scope·ifLet·forEach 조합 | [자식 Feature 연결](references/child-features.md) |
| sheet 표시, 개별 @Presents와 Destination 선택, self-dismiss와 상위 처리 | [화면 표시](references/presentation.md) |
| 같은 부모의 상호 배타적인 복수 표시 대상 | [Destination 구성](references/destination.md) |
| Alert·ConfirmationDialog와 버튼 이벤트 | [확인·안내 표시](references/alerts.md) |
| NavigationStack·StackState·Path | [스택 화면 전환](references/stack-navigation.md) |
| TextField·Toggle 바인딩과 변경 후 추가 작업 | [바인딩](references/bindings.md) |
| 비동기 호출·취소·debounce, 화면 진입 로딩·새로고침 | [Effect 처리](references/effects.md) |
| Client 선언·실제 구현·등록 | [Dependency Client](references/dependency-clients.md) |

아직 정하지 않은 세부 이벤트 네이밍은 기존 프로젝트 패턴을 따른다.

기준 버전 표시는 이 스킬의 작성 기준을 알리기 위한 정보다. 스킬을 사용할 때마다 최신 릴리스, SHA 또는 버전 간 diff를 조회하지 않는다. 실제 구현에서 API 호환성 문제가 발생하거나 확인이 필요할 때만 관련 정보를 확인한다.
