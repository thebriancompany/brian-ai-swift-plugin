# TCA 버전 기준

이 문서는 스킬 작성자가 사용하는 버전 근거 기록이다. 일반 스킬 사용 시에는 읽지 않는다. 사용자에게 보여주는 기준 버전은 [SKILL.md](../../skills/brian-tca-feature/SKILL.md)에 표시하며, 기준을 변경할 때 이 기록과 함께 갱신한다.

| 항목 | 값 |
| --- | --- |
| 저장소 | https://github.com/pointfreeco/swift-composable-architecture |
| 채택 버전 / 태그 | `1.26.2` |
| 확인한 커밋 SHA | `377da4061db10d26337a71bb279c506bb951f50f` |
| 기준 확인일 | 2026-10-26 (Asia/Seoul) |
| 정책 | 고정 버전을 사용하고, 명시적인 업데이트 작업에서 diff를 검토한 뒤 변경한다. |

## 근거

- [공식 릴리스](https://github.com/pointfreeco/swift-composable-architecture/releases/tag/1.26.2)
- [기준 커밋](https://github.com/pointfreeco/swift-composable-architecture/commit/377da4061db10d26337a71bb279c506bb951f50f)
- [기준 소스 트리](https://github.com/pointfreeco/swift-composable-architecture/tree/377da4061db10d26337a71bb279c506bb951f50f)
- [기본 패키지 설정](https://github.com/pointfreeco/swift-composable-architecture/blob/377da4061db10d26337a71bb279c506bb951f50f/Package.swift)

사용자가 지정한 버전과 해당 커밋을 확인했다. 현재 최신 릴리스 여부를 지속적으로 보증하는 기록은 아니다.

## 고정의 의미

- 고정 대상은 이 스킬의 API 판단과 예제 검토 기준이다. 대상 앱의 패키지 설정을 자동으로 변경하는 정책이 아니다.
- 새 버전이 나왔다는 이유만으로 기준을 이동하지 않는다. 업데이트 절차를 완료한 뒤 이 표를 갱신한다.
- Swift, Xcode, 배포 대상 및 전이 의존성의 버전은 TCA 버전과 구분한다. 대상 프로젝트와 기준 커밋의 해당 설정을 확인한다.
- 기본 Package.swift 하나만으로 모든 도구 체인의 요구사항을 단정하지 않는다. 기준 커밋에는 `Package@swift-6.1.swift` 같은 버전별 manifest도 있으므로 적용되는 파일을 확인한다.

## 초기 확인 범위

초기에는 릴리스 노트, 태그의 커밋 및 기본 패키지 설정을 확인했다. 이후 추가한 규칙과 예제의 API 근거·검토 범위는 각 참고 문서에 기록한다. 전체 예제의 컴파일 검증은 아직 수행하지 않았다.

## Dependency Client 근거 확인

문서 일관성 점검에서 Client 선언과 검색 호출의 인자 표기, 환경별 기본값을 대조했다. TCA 기준은 변경하지 않았다.

- 기준 커밋의 [Package.resolved](https://github.com/pointfreeco/swift-composable-architecture/blob/377da4061db10d26337a71bb279c506bb951f50f/Package.resolved)에 기록된 swift-dependencies는 `1.13.0`, 커밋은 `f80552807ec92f72fe3fe4543d71879182b0bfd5`다.
- [DependencyClient 매크로 설명](https://github.com/pointfreeco/swift-dependencies/blob/f80552807ec92f72fe3fe4543d71879182b0bfd5/Sources/DependenciesMacros/Macros.swift)에서 이름 있는 인자 메서드 생성을 확인했다.
- [DependencyKey 정의](https://github.com/pointfreeco/swift-dependencies/blob/f80552807ec92f72fe3fe4543d71879182b0bfd5/Sources/Dependencies/DependencyKey.swift)에서 TestDependencyKey 상속 및 previewValue 기본 구현을 확인했다.

이 기록은 확인에 사용한 소스이며, 대상 앱의 전이 의존성 버전 고정이나 컴파일 검증을 의미하지 않는다.
