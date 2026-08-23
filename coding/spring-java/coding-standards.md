# 코딩 표준 — Java · Spring

> 언어 무관 원칙은 [common/design-principles.md](../common/design-principles.md).
> 프로젝트마다 답이 갈리는 항목은 [_프로젝트에서_정할_것.md](../_프로젝트에서_정할_것.md).

## 자료형 — `record` vs `class`

| 용도 | 타입 | 이유 |
|---|---|---|
| ORM 매핑 대상(엔티티) | `class` | no-arg 생성자·가변 필드·프록시가 필요 |
| DTO · VO · 검색조건 · 응답 모델 | `record` | 불변, `equals`/`hashCode`/`toString` 자동, 의도가 드러남 |

> 언어 레벨이 `record` 를 지원하지 않거나 하위 호환이 필요하면 전부 `class` 로 간다.
> 이 선택은 프로젝트에서 정한다.

## 경로·파일 I/O

- 경로 조합은 `Path.of()` 를 쓴다. 문자열 `+`, `File.separator`, `Paths.get()` 금지.
- `java.nio.file.Files` 를 우선한다. 레거시 `java.io.File` 은 지양한다.
- DB 에 저장하는 경로는 구분자를 정규화(`\`→`/`)한 절대경로로 통일해 OS 종속을 없앤다.

## 트랜잭션

- CUD 메서드는 `@Transactional` 필수, 조회는 `@Transactional(readOnly = true)`.
- 파일 I/O 와 외부 API 호출은 트랜잭션 밖에서 수행한다 →
  [file-io-transaction.md](file-io-transaction.md)

## 로깅

- 로거는 lombok `@Slf4j` 로 통일한다. `private static final Logger log = LoggerFactory.getLogger(...)`
  수동 선언을 쓰지 않는다 — 필드명·대상 클래스 오기를 막고 보일러플레이트를 없앤다.
  거동은 같다(둘 다 slf4j `Logger`).
- `org.slf4j.Logger`/`LoggerFactory` 를 직접 import 하지 않는다. `MDC` 는 예외(상관관계 키 주입).
- 예외 스택트레이스는 항상 `log.error(msg, e)` 로 남긴다 → [exception-handling.md](exception-handling.md)

## 의존성 주입

- **생성자 주입만 쓴다.** 필드 주입(`@Autowired`) 금지. 필드는 `private final`.
- 기본은 lombok `@RequiredArgsConstructor`. 단일 생성자에 `@Autowired` 는 붙이지 않는다
  (Spring 4.3+ 는 자동 주입한다).
- 명시적 생성자는 아래 둘에만 쓴다.
  1. 다중 생성자가 필요할 때(테스트 시드용 보조 생성자 등)
  2. 생성자 안에서 변환·검증이 필요할 때(`new TransactionTemplate(txManager)` 등)
- `@RequiredArgsConstructor` 는 필드 선언 순서로 생성자를 만든다. 위 예외에 해당하면
  명시적 생성자가 그 암묵 결합을 없애 오히려 명확하다.

## 설정 바인딩

- **모든 외부 설정은 `@ConfigurationProperties` 로 바인딩한다. `@Value` 를 쓰지 않는다.**
  예외절을 두지 않는 것이 규칙이다 — 설정을 읽는 방식을 하나로 고정해야 추적이 된다.
- **prefix 단위로 타입 하나.** 관련 키는 한 prefix 아래 중첩 타입으로 모은다.
  키가 하나뿐이어도 타입으로 묶는다.
- 기본값·검증은 바인딩 타입의 생성자에서 처리한다.
- **`blank` 와 `absent` 는 다르다.** 기본값은 키가 **없을 때만** 적용된다. 키가 있고 값이 빈
  문자열이면 `""` 가 그대로 바인딩된다. "비어 있으면 기본값" 로직은 생성자에서 직접 처리한다.
- 시크릿(`password`·`secret`·`key`·`token`)을 가진 타입은 `toString()` 을 재정의해 마스킹한다.
  설정 엔드포인트로 노출될 수 있다.
- 설정을 생성자 인자에서 빼고 Properties 빈 주입으로 바꾸면 생성자가 순수 DI 가 되어
  `@RequiredArgsConstructor` 를 쓸 수 있다.

## Javadoc

| 대상 | 강제 수준 |
|---|---|
| public 타입(class·interface·enum·record) | **필수.** 최소 역할 한 문장 |
| public/protected 메서드 | 시그니처로 자명하면 생략. 비자명한 제약이 있으면 `@param`/`@return`/`@throws` 로 명시 |
| 인터페이스 메서드 | **필수.** 구현 교체를 전제한 계약은 구현체가 아니라 인터페이스가 단일 출처다 |
| private·패키지 내부 | 자명하면 생략. 비자명한 로직만 "왜"를 한 줄 |

- **재진술 금지.** "무엇을 하는가"를 시그니처로 반복하지 말고 **계약·제약·부수효과·이유**를 적는다.
  DI 생성자에 "의존성을 주입한다"고 적는 것은 정보량이 0 이다.
- 코드 식별자는 `{@code}`, 타입·멤버 링크는 `{@link}`.
- 입력에서 결과가 한눈에 안 드러나는 public 메서드는 호출식 → 결과 형태를 한 줄 예제로 붙인다.
- 예외 계약을 명시한다. 파일 I/O 를 포함한 메서드는 `@Transactional` 금지 경고를 Javadoc 에 남긴다.

## 관측성

관측 코드(필터·메트릭·트레이싱)는 별도 패키지에 격리한다. 업무 코드와 섞지 않는다.
