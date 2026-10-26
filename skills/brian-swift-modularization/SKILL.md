---
name: brian-swift-modularization
description: Brian의 개인 컨벤션에 맞춰 로컬 SPM 패키지와 타깃, 모듈 간 의존성, Core 재노출, 공유 데이터와 API 배치를 작성·수정한다. FeatureCore·ClientCore를 사용하는 프로젝트에서 Feature·Client 파일을 추가하거나 import를 수정할 때도 적용한다. Feature 내부 상태·이벤트·로직 구현 규칙은 다루지 않는다.
---

# Brian Swift Modularization

## 적용 범위

- 로컬 SPM 패키지·타깃 추가, 기존 코드의 모듈 추출, 모듈 간 의존성과 공개 범위를 변경할 때 적용한다.
- FeatureCore·ClientCore를 사용하는 프로젝트에서 Feature·Client 파일을 추가하거나 import를 수정할 때는 Core import 구성에 한해 적용한다.
- TCA Feature 내부 구현 규칙과 테스트 작성은 이 스킬의 범위에 포함하지 않는다.

## 공통 원칙

- 사용자 요청과 대상 프로젝트의 명시적인 규칙을 존중한다. 기존 구조를 확인하지 않고 전체 프로젝트를 재구성하지 않는다.
- 새 패키지는 `swift-tools-version: 6.4`를 사용한다. 기존 프로젝트의 도구 버전·배포 대상은 모듈화만을 위해 일괄 변경하지 않는다.
- TCA 버전 숫자는 이 스킬에서 별도로 관리하지 않는다. 새 요구사항의 기준이 필요할 때만 버전 선택 절차를 통해 TCA 스킬의 현재 기준을 참조한다.
- 인증·캐시·DB 엔진 등 확정되지 않은 제품 동작은 기존 구현을 우선한다. 작업에 반드시 필요한 결정이 없으면 사용자에게 묻는다.

## 작성·수정

1. 프로젝트 지침과 대상 코드·인접 구현을 확인한다. 변경에 관련된 `Package.swift`, Core 재노출, 앱 연결을 확인한다.
2. 아래 표에서 작업에 해당하는 참고 자료만 읽고 적용한다. 소스·import·타깃 의존성·제품 노출·앱 연결 중 변경에 필요한 항목을 함께 수정한다.
3. 대상 프로젝트의 툴체인과 플랫폼에 맞춰 관련 빌드·진단을 확인한다. manifest를 변경했다면 평가도 확인하되, 이를 소스·앱 빌드 성공으로 설명하지 않는다. 확인하지 못한 부분은 별도로 알린다.

| 작업 | 참고 자료 |
| --- | --- |
| 패키지·타깃 경계, 의존 방향, 공개 범위 결정 | [구조와 공통 의존성](references/conventions.md) |
| Core 재노출 또는 Feature·Client의 import 구성 | [Core와 Exports.swift](references/conventions.md#core와-exportsswift) |
| Entity·API·로컬 저장소 배치 | [데이터와 외부 접근](references/data-access.md) |
| 패키지·타깃 생성, manifest 작성·수정, 앱 연결 | [SPM 생성과 연결](references/spm-setup.md) |
| 기존 코드를 다른 모듈로 이동 | [모듈 추출 절차](references/spm-setup.md#기존-코드를-모듈로-추출할-때) |
| 외부 패키지 요구사항의 버전 선택 | [외부 의존성 버전 선택](references/spm-setup.md#외부-의존성-버전-선택) |

Core import만 조정할 때는 해당 항목만 적용한다. 이 작업만을 위해 패키지 구조를 변경하거나 버전·API 배치 문서를 읽지 않는다. 새로운 타깃 의존성이 필요하면 의존 방향과 SPM 연결 항목도 읽는다. 여러 종류의 변경이 함께 있으면 해당하는 항목을 조합한다.
