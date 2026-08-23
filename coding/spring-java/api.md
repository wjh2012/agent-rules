# API 컨트롤러 · OpenAPI — Java · Spring

> 허용 HTTP 메서드·URI 정책·성공 응답 형태는 프로젝트마다 다르다 →
> [_프로젝트에서_정할_것.md](../_프로젝트에서_정할_것.md)

## 1. 컨트롤러 문서화

- **클래스에 `@Tag(name, description)` 필수.** `name` 은 문서 UI 의 그룹 제목, `description` 은 한 문장 설명.
- **핸들러마다 `@Operation(summary)` 필수.** 동작이 비자명하거나 제약(허용 확장자·정렬 기본값 등)이
  있으면 `description` 을 더한다.

## 2. 메서드명은 프로젝트 전역에서 고유해야 한다

문서 생성기가 메서드명을 `operationId` 기본값으로 쓴다. 이름이 겹치면 `upload_1` 같은 임의 접미어가
붙어 클라이언트 코드 생성 결과가 흔들린다.

- 반드시 **도메인 풀네임을 접두어로** 포함한다. 약어를 쓰지 않는다.
- 조회는 `get` 으로 통일하고 **대상 명사의 단수·복수로** 단건·다건을 구분한다
  (`getUserDetail` / `getUsers`).
- 동사는 하나로 통일한다 — `create`·`delete` 를 쓰고 `add`·`remove` 를 섞지 않는다.
- 상세: [naming.md](naming.md)

## 3. 요청·응답 DTO

- API 계층 DTO 는 `record` 로 둔다([coding-standards.md](coding-standards.md)).
- **클래스에 `@Schema(description)`, 필드마다 `@Schema(description, example)`** 를 단다.
- **`example` 은 그 필드의 의미에 맞는 실제 형태값**을 쓴다. 다른 필드 값을 복사해 붙이지 않는다
  (날짜 필드에 ID 예시를 넣지 않는다). 빈 문자열 example 은 선택값일 때만.
- `description` 오타에 주의한다. 문서 UI 에 그대로 노출된다.
- 엔티티·조회 결과에서 응답 DTO 로 바꾸는 정적 팩토리 `from(...)` 을 둔다.

## 4. 요청 검증

- 요청 DTO 필드에 제약 애너테이션(`@NotBlank`·`@NotNull`·`@Size` 등)을 단다.
- 핸들러 인자에 `@Valid` 를 붙여 검증을 발동한다.
- **검증 실패는 전역 핸들러가 400 으로 변환한다.** 컨트롤러에서 직접 검사·분기하지 않는다
  → [exception-handling.md](exception-handling.md)
- 검색 조건처럼 검증이 없는 DTO 에는 `@Valid` 를 생략한다.
- `@Size(max = N)` 의 N 은 **문자 수**다. DB 컬럼이 byte 길이면 그 의미차를 주석으로 못박는다.

## 5. 파라미터 바인딩

| 상황 | 방식 |
|---|---|
| JSON 본문 | `@Valid @RequestBody DTO` |
| 멀티파트 + 폼 필드 | `@RequestPart` + `@Valid @ParameterObject @ModelAttribute DTO` |
| 쿼리 파라미터 | `@ParameterObject @ModelAttribute DTO` — 문서에 필드별 입력란이 생긴다 |
| 페이징 | `@ParameterObject @PageableDefault(...) Pageable` |
| 경로 변수 | `@PathVariable` |

- 단일 파라미터도 조건 DTO 로 감싼다. `@RequestParam` 을 흩뿌리지 않는다.
- 멀티파트 타입은 **컨트롤러 계층에서만** 다룬다. 서비스에는 메타 DTO 나 `InputStream` 으로
  변환해 넘긴다.
