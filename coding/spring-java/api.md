# API 규칙 — Java · Spring

> `BaseResponse` 같은 공통 응답 래퍼의 실제 클래스명은 프로젝트 `CLAUDE.md` 에서 바인딩한다.

## 1. DTO 규칙

- DTO는 `@Data`로 통일한다. `@Getter @Setter` 개별 사용 금지.
- **Builder 없는 DTO**: `@Data @NoArgsConstructor @AllArgsConstructor`
- **Builder 있는 DTO**: `@Data @Builder @NoArgsConstructor @AllArgsConstructor`
- `import lombok.*;` 금지 — 개별 import 사용.

## 2. API 네이밍 규칙

- **URI**: 소문자와 하이픈(`-`) 사용, 복수형 명사 사용 (예: `/api/items`, `/api/users`)
- **JSON Field**: 카멜 케이스(camelCase)

### Controller 메서드 선언 순서

CRUD 컨트롤러의 메서드는 다음 순서로 선언한다:

1. **Create** — 생성
2. **Read** — 목록 조회 → 단건 조회 → 검색/중복검증 등 보조 조회
3. **Update** — 수정
4. **Delete** — 삭제
5. **기타** — 비CRUD 액션 (approve, reject, download 등)

> 인증·배치 제어 등 CRUD 패턴이 아닌 컨트롤러는 이 규칙을 적용하지 않는다.

### Controller 메서드 네이밍

조회는 `get`으로 통일하고, 대상 명사로 단건/다건을 구분합니다.

| 패턴 | 용도 | 예시 |
|------|------|------|
| `get{도메인}Detail` | 단건 조회 | `getTaskDetail`, `getUserDetail` |
| `get{도메인}s` | 다건 조회 (목록/검색/집계) | `getTasks`, `getUsers` |
| `create~` | 리소스 생성 | `createTask`, `createUser` |
| `update~` | 리소스 수정 | `updateTask`, `updateUserRole` |
| `delete~` | 리소스 삭제 | `deleteTask`, `deleteUser` |
| `download~` | 파일/바이너리 반환 (`ResponseEntity<Resource>`) | `downloadReport` |

**전역 고유 이름 원칙:**
- 메서드명은 **프로젝트 전체에서 고유**해야 한다. 코드 생성기에서 `getDetail1`, `getDetail2`로 변환되는 것을 방지한다.
- 반드시 **도메인 풀네임**을 접두어로 포함한다. 약어 대신 전체 이름을 사용한다.
- 동사는 `create`, `delete`로 통일한다. (`add`, `remove` 등 동의어 대신)

### HTTP 메서드 정책

허용 HTTP 메서드는 **프로젝트에서 정한다.** 조직 보안 정책으로 POST·GET 만 허용하는
경우가 있으므로, 허용 메서드와 CUD 경로 패턴을 프로젝트 `CLAUDE.md` 에 적고 그것을 따른다.

- 벌크 작업은 단건 경로(`/{id}/...`)를 반복하지 않고, 별도 엔드포인트에
  `@Valid @RequestBody` 로 ID 목록 DTO 를 받는다.
- 중복검증 등 조건 조회는 `GET` + `@ParameterObject @ModelAttribute` 로 받는다.

### 비CRUD 도메인 메서드명 예외

Auth 등 CRUD 패턴이 아닌 인증 행위 도메인은 행위동사 + 도메인 접미어를 허용한다:

| 메서드명 | 용도 |
|---------|------|
| `loginAuth` | 로그인 |
| `logoutAuth` | 로그아웃 |
| `registerAuth` | 회원가입 |
| `refreshAuth` | 토큰 재발급 |

## 3. Controller 파라미터 바인딩 규칙

| 상황 | 어노테이션 | 비고 |
|------|-----------|------|
| JSON body (POST) | `@Valid @RequestBody DTO` | CUD 요청에 사용 |
| Multipart + 폼 필드 (POST) | `@RequestPart("file")` + `@Valid @ParameterObject @ModelAttribute DTO` | 파일 업로드 시 사용 |
| 쿼리 파라미터 (GET) | `@ParameterObject @ModelAttribute DTO` | Swagger에서 필드별 개별 입력란 표시 |
| 페이징 (GET) | `@ParameterObject @PageableDefault(...) Pageable` | 목록 조회 시 사용 |
| 경로 변수 | `@PathVariable` | 리소스 ID 등 |

- **`@RequestParam` 사용 금지** — 단일 파라미터도 Condition/Request DTO로 감싸서 `@ModelAttribute` 또는 `@RequestBody`로 받는다.
- POST 요청의 파라미터는 **body**로 받는다.
- GET 요청의 파라미터는 **Condition DTO** + `@ParameterObject @ModelAttribute`로 받는다.
- `@Valid`는 Bean Validation 어노테이션이 있는 DTO에 반드시 붙인다. 검색 조건(SearchCondition)처럼 검증 없는 DTO에는 생략한다.

## 4. DTO 상세 규칙

### Request DTO

- `@Data @NoArgsConstructor @AllArgsConstructor` (Builder 사용 안 함)
- 필드에 `@Schema(description, example)` 어노테이션을 붙인다.
- Jakarta Validation 어노테이션으로 필수값을 검증한다: `@NotBlank`, `@NotNull`, `@Size` 등.

```java
@Schema(description = "항목 생성 요청")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class ItemCreateRequest {

    @Schema(description = "항목 코드", example = "ITEM-001")
    @NotBlank
    private String code;

    @Schema(description = "항목명", example = "샘플 항목")
    @NotBlank
    private String name;

    @Schema(description = "설명", example = "설명 텍스트")
    private String description;
}
```

### Response DTO

- `@Data @Builder @NoArgsConstructor @AllArgsConstructor`
- 필드에 `@Schema(description, example)` 어노테이션을 붙인다.
- Entity → DTO 변환은 **`from(Entity)` 정적 팩토리 메서드**를 사용한다.
- 단순 값 래핑은 `of(...)` 메서드를 사용한다.

```java
@Schema(description = "항목 응답")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ItemResponse {

    @Schema(description = "항목 ID", example = "550e8400-e29b-41d4-a716-446655440000")
    private String id;

    @Schema(description = "항목명", example = "샘플 항목")
    private String name;

    public static ItemResponse from(Item item) {
        return ItemResponse.builder()
                .id(item.getId())
                .name(item.getName())
                .build();
    }
}
```

### SearchCondition DTO

- `@Data @NoArgsConstructor @AllArgsConstructor`
- 필드에 `@Schema(description, example)` 어노테이션을 붙인다.
- 검색 조건은 선택값이므로 Validation 어노테이션을 붙이지 않는다.
- Controller에서 `@ParameterObject @ModelAttribute`로 바인딩한다.

## 5. API 반환 규칙

- API 응답은 **`BaseResponse<T>`로 통일**한다.
- Controller는 원시 DTO, `List<DTO>`, `Map` 등을 직접 반환하지 않는다.
- 단건/다건/상태 응답 모두 `BaseResponse<T>`에 담아 반환한다.
- **예외**: 파일 다운로드, 바이너리 스트림 응답은 `ResponseEntity<Resource>`를 사용한다.

### CUD 응답 원칙

CUD API는 `BaseResponse<Void>`를 사용하지 않는다. 처리 결과를 클라이언트에 반환한다.

| 작업 | 반환 내용 | 예시 |
|------|----------|------|
| 생성 | 생성된 리소스 ID | `BaseResponse<String>` 또는 전용 Response DTO |
| 수정 | 수정된 리소스 ID | `BaseResponse<String>` 또는 전용 Response DTO |
| 삭제 | 삭제된 리소스 ID(목록) | `BaseResponse<String>` 또는 `BaseResponse<List<String>>` |

## 6. MultipartFile 규칙

- `MultipartFile`은 **컨트롤러 계층에서만** 사용합니다.
- 서비스에는 파일 메타 DTO 또는 `InputStream`으로 변환하여 전달합니다.
- 변환 로직은 컨트롤러의 private 헬퍼 메서드로 구현합니다.

### 예시

```java
@GetMapping("/api/tasks/{taskId}")
public BaseResponse<TaskDetailResponse> getTaskDetail(@PathVariable String taskId) {
    TaskDetailResponse response = taskService.getTaskDetail(taskId);
    return BaseResponse.success(response);
}

@GetMapping("/api/tasks")
public BaseResponse<List<TaskResponse>> getTasks(TaskSearchRequest request) {
    List<TaskResponse> response = taskService.getTasks(request);
    return BaseResponse.success(response);
}

@PostMapping("/api/tasks")
public BaseResponse<TaskCreateResponse> createTask(@RequestBody TaskCreateRequest request) {
    TaskCreateResponse response = taskService.createTask(request);
    return BaseResponse.success(response);
}