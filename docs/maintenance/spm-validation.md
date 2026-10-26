# SPM 생성 예시 검증

- 확인일: 2026-10-26 (Asia/Seoul)
- 대상: `skills/brian-swift-modularization/references/spm-setup.md`의 Shared·Clients·Features manifest
- 환경: Apple Swift 6.4 (`swiftlang-6.4.0.34.1`), arm64 macOS
- 방법: 문서의 Swift 코드 블록을 추출하고 버전 자리표시자를 TCA 스킬 기준 및 기존 버전 근거의 swift-dependencies 해결 버전으로 치환한 뒤, 임시 디렉터리의 세 `Package.swift`에 대해 각각 `swift package dump-package` 실행
- 결과: 세 manifest 평가 성공. 출력에서 로컬 패키지 식별자, product·target 및 의존 선언을 확인했다.
- 실행 환경 조정: 샌드박스의 기본 캐시 경로 쓰기 제한으로 `CLANG_MODULE_CACHE_PATH`를 임시 디렉터리로 지정하고 `--disable-sandbox`를 사용했다. 사용자 캐시 비활성화 경고가 있었으나 평가 종료 코드는 모두 0이었다.
- 미검증: 원격 의존성 해결, 실제 소스·Core 재노출·매크로 컴파일, Xcode 앱 제품 연결 및 플랫폼 빌드

TCA 검증 값은 [TCA Feature 스킬](../../skills/brian-tca-feature/SKILL.md)의 현재 기준을, swift-dependencies 검증 값은 기존 [버전 근거](version-baseline.md)의 해결 버전을 사용했다. 모듈화 문서는 버전 숫자를 보관하지 않는다. `from:` 요구사항은 최소 버전이며 실제 해결 버전을 고정하지 않는다. Swift tools 6.4는 사용자가 정한 새 패키지 기본값이다. iOS 17은 설명용 설정이며 사용자 프로젝트에 강제하지 않는다.
