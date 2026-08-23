# 코딩 표준 — Java · Spring

> 언어 무관 원칙은 [common/design-principles.md](../common/design-principles.md) 참조.

## 자료형·API

- **자료형**: `record` 대신 `class` 를 사용한다. (하위 Java 호환성)
- **경로 조합**: `Path.of()` 를 사용한다. (문자열 연결 `+`, `File.separator`, `Paths.get()` 금지)

## 예외 처리

- 비즈니스 예외는 **프로젝트 공통 베이스 예외를 상속한 도메인 예외**로 던진다.
  (예: `{Domain}Exception extends BusinessException`)
- 공통 런타임 예외를 직접 쓰지 않는다.
- 전역 예외 핸들러에서 통합 처리한다. **Controller 에서 catch 하지 않는다.**
- 파일 I/O 예외는 전용 예외 타입으로 구분한다.

> 베이스 예외·핸들러의 실제 클래스명은 프로젝트 `CLAUDE.md` 에서 바인딩한다.

## 트랜잭션

- CUD 메서드는 `@Transactional` 필수. 조회는 `@Transactional(readOnly = true)`.
- 외부 API 호출과 파일 I/O 는 트랜잭션 밖에서 수행한다.

> 상세: [file-io-transaction.md](file-io-transaction.md)

## 패키지 구조

도메인 중심 계층형 (`domain.{도메인}.api/service/repository/entity/dto`).
의존성은 `api → service → repository → entity` 단방향.

> 상세: [package-structure.md](package-structure.md)

## 파일 처리

업무 도메인에서 파일을 다룰 때는 **이력을 남기는 단일 진입점을 경유한다.**
도메인 코드가 스토리지 API 를 직접 호출하지 않는다.

> 진입점 클래스명은 프로젝트 `CLAUDE.md` 에서 바인딩한다.
