# 파일 I/O 트랜잭션 분리 규칙 — Java · Spring

> 원칙만 필요하면 [common/design-principles.md](../common/design-principles.md) 참조.

## 1. 원칙

**파일 I/O(업로드/다운로드)는 DB 트랜잭션 밖에서 수행한다.**

파일 저장·읽기는 디스크 또는 네트워크 I/O로 수백 ms~수 초가 소요될 수 있다.
이 시간 동안 `@Transactional`이 열려 있으면 DB 커넥션이 아무 일도 하지 않고 점유되어, 커넥션 풀(기본 HikariCP 10개) 고갈 → 전체 API 대기 상태를 유발한다.

```java
// Bad — 파일 I/O가 트랜잭션 안에 있음
@Transactional
public void createSomething(Request req, List<MultipartFile> files) {
    String bundleId = fileService.upload(dir, files);  // 파일 I/O
    someMapper.insertSomething(entity);  // DB
}

// Good — 파일 I/O를 트랜잭션 밖에서 수행, TransactionTemplate으로 DB만 트랜잭션
public void createSomething(Request req, List<MultipartFile> files) {
    String bundleId = uploadFilesIfPresent(files);  // 파일 I/O (트랜잭션 없음)

    transactionTemplate.execute(status -> {
        someMapper.insertSomething(entity);  // DB만 트랜잭션
        return null;
    });
}
```

---

## 2. 파일 처리 진입점에 `@Transactional` 을 걸지 않는다

파일 저장과 메타데이터 기록을 함께 하는 진입점 메서드에는 `@Transactional` 을 선언하지 않는다.
내부의 DB 처리 컴포넌트가 자체 `@Transactional` 로 DB 작업만 짧게 처리하도록 둔다.

```
upload() — 트랜잭션 없음
├─ storage.save() × N          ← 파일 I/O
├─ 메타데이터 저장              ← 자체 @Transactional
└─ 이력 기록                    ← 자체 @Transactional
```

**호출부에서 `@Transactional` 로 감싸면 이 설계가 무효화된다.**
내부 컴포넌트의 `@Transactional(REQUIRED)` 이 외부 트랜잭션에 참여하게 되어,
파일 I/O 동안 DB 커넥션이 점유된다.

---

## 3. 적용 패턴 — `TransactionTemplate`

파일 I/O가 포함된 메서드는 `@Transactional`을 선언하지 않고, `TransactionTemplate`으로 DB 작업만 감싼다.
public 메서드가 하나로 유지되어 API가 명확하고, self-injection이 불필요하다.

### 3.1 생성(create)

```java
@Service
@RequiredArgsConstructor
public class SomeService {

    private final FileService fileService;
    private final SomeMapper someMapper;
    private final TransactionTemplate transactionTemplate;

    public SomeResponse create(SomeRequest request, List<MultipartFile> files) {
        // 1. 파일 I/O — 트랜잭션 밖
        String bundleId = uploadFilesIfPresent(files);

        // 2. DB 저장 — 트랜잭션 안
        return transactionTemplate.execute(status -> {
            SomeEntity entity = SomeEntity.builder()
                    .attachmentId(bundleId)
                    .build();
            someMapper.insertSomething(entity);
            return SomeResponse.from(entity);
        });
    }
}
```

### 3.2 수정(update) — 명시적 update 필수

```java
public SomeResponse update(String id, SomeRequest request, List<MultipartFile> files) {
    // 1. 파일 I/O — 트랜잭션 밖
    String bundleId = uploadFilesIfPresent(files);

    // 2. 엔티티 조회 + 수정 + 저장 — 트랜잭션 안 (명시적 update 호출)
    return transactionTemplate.execute(status -> {
        SomeEntity entity = findOrThrow(id);
        entity.update(..., bundleId);
        someMapper.updateSomething(entity);
        return SomeResponse.from(entity);
    });
}
```

### 3.3 다운로드 — 트랜잭션 불필요

파일 읽기 I/O는 트랜잭션이 필요 없다.
개별 Repository 호출(`findById`, `save`)은 각각 독립적인 트랜잭션으로 처리된다.

```java
// @Transactional 선언 불필요
public FileDownloadInfo getDownloadInfo(String id, String fileEntryId) {
    findOrThrow(id);
    return fileService.download(fileEntryId);
}
```

### 3.4 파일 I/O가 없는 CUD — `@Transactional` 유지

파일 I/O 없이 순수 DB 작업만 수행하는 메서드는 `@Transactional`을 그대로 사용한다.

```java
@Transactional
public SomeResponse delete(String id) {
    SomeEntity entity = findOrThrow(id);
    entity.delete();
    return SomeResponse.from(entity);
}
```

---

## 4. `@Transactional` 중첩 금지

파일 I/O를 포함하는 메서드에 `@Transactional`을 선언하면, 내부 `TransactionTemplate`이 외부 트랜잭션에 참여하여 분리 설계가 무효화된다.

```java
// Bad — @Transactional이 파일 I/O를 감싸고, TransactionTemplate은 외부 tx에 참여
@Transactional  // ← 절대 금지
public SomeResponse create(SomeRequest request, List<MultipartFile> files) {
    String bundleId = uploadFilesIfPresent(files);  // 파일 I/O (DB 커넥션 점유!)
    return transactionTemplate.execute(status -> {  // 외부 tx에 참여 → 분리 효과 없음
        // ...
    });
}
```

`REQUIRES_NEW`로 내부 트랜잭션을 강제 분리해도 해결되지 않는다:
- 외부 `@Transactional`의 DB 커넥션은 파일 I/O 동안 **여전히 점유**
- 내부 `REQUIRES_NEW`가 **추가 커넥션을 획득**하여 풀 고갈이 더 빨라짐
- 내부 트랜잭션이 커밋된 후 외부에서 예외 발생 시 **롤백 불가**

**방어 방법**: 파일 I/O를 포함하는 메서드에는 Javadoc으로 `@Transactional` 금지를 명시한다.

```java
/**
 * <b>주의: 이 메서드에 @Transactional을 선언하지 말 것.</b>
 * 파일 I/O 동안 DB 커넥션이 점유되어 커넥션 풀 고갈을 유발한다.
 * 상세: docs/rules/file-io-transaction-rule.md
 */
public SomeResponse create(SomeRequest request, List<MultipartFile> files) {
    // ...
}
```

---

## 5. 클래스 레벨 `@Transactional` 사용 시 주의

클래스 레벨에 `@Transactional(readOnly = true)`를 선언하면, 파일 I/O 오케스트레이션 메서드에도 읽기 트랜잭션이 열린다.
이 경우 오케스트레이션 메서드에서도 DB 커넥션이 점유되므로, **메서드 레벨에서 개별 선언하는 것을 권장**한다.

```java
// Bad — 클래스 레벨 트랜잭션이 파일 I/O 메서드에도 적용됨
@Service
@Transactional(readOnly = true)
public class SomeService {
    public SomeResponse create(...) { ... }  // readOnly 트랜잭션이 열림
}

// Good — 메서드별 명시 선언
@Service
public class SomeService {
    @Transactional(readOnly = true)
    public Page<SomeResponse> getList(...) { ... }

    @Transactional
    public SomeResponse delete(...) { ... }

    // 파일 I/O 포함 — @Transactional 없음, 내부에서 TransactionTemplate 사용
    public SomeResponse create(...) { ... }
}
```

---

## 6. 체크리스트

- [ ] `FileService.upload()` 호출이 `@Transactional` 밖에 있는가
- [ ] `FileService.download()` 호출이 `@Transactional` 밖에 있는가
- [ ] 파일 I/O 포함 메서드에서 `TransactionTemplate`으로 DB 작업만 감싸고 있는가
- [ ] 파일 I/O 포함 메서드에 `@Transactional`이 선언되어 있지 않은가 (클래스/메서드 레벨 모두)
- [ ] 파일 I/O 포함 메서드의 Javadoc에 `@Transactional` 금지 경고가 명시되어 있는가
- [ ] 클래스 레벨 `@Transactional(readOnly = true)` 사용 시, 파일 I/O 메서드가 영향받지 않는가
- [ ] 순수 DB 작업만 수행하는 CUD 메서드(`delete` 등)는 `@Transactional`을 유지하는가
- [ ] 조회 메서드는 `@Transactional(readOnly = true)`를 명시하는가
