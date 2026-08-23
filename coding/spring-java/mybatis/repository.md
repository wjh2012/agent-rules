# Repository 규칙 — MyBatis

> **MyBatis 전용이다.** JPA 프로젝트에는 적용하지 않는다.

MyBatis 기반 Mapper 직접 사용 패턴을 준수합니다. **Mapper 인터페이스·XML을 항상 동기화**하는 것이 핵심입니다.

## 1. 구조

Service가 MyBatis Mapper를 직접 주입받아 사용합니다.

```
{도메인}/repository/
└── {Entity}Mapper.java              # @Mapper 인터페이스
```

**Mapper XML 위치**: `src/main/resources/mapper/{도메인명}/{Entity}Mapper.xml`

**Service에서 Mapper 직접 사용**:

```java
@Service
@RequiredArgsConstructor
public class SampleService {

    private final SampleMapper sampleMapper;

    // ...
}
```

## 2. MyBatis Mapper 표준

### Mapper 인터페이스

```java
@Mapper
public interface {Entity}Mapper {

    Optional<{Entity}> select{Entity}ById(@Param("{id}") String {id});

    void insert{Entity}({Entity} entity);

    void update{Entity}({Entity} entity);

    void delete{Entity}ById(@Param("{id}") String {id});
}
```

### Mapper XML resultMap

- Entity의 **모든 필드**를 `<resultMap>`에 매핑한다.
- `BaseEntity` 공통 감사 필드(`createdBy`, `createdAt`, `updatedBy`, `updatedAt`)도 반드시 포함한다.
- `namespace`는 Mapper 인터페이스의 FQCN과 정확히 일치시킨다.

## 3. insert/update 명시적 호출

Service에서 insert/update를 명시적으로 구분하여 호출한다. JPA의 `save()` 추상화는 사용하지 않는다.

```java
// 신규 생성
@Transactional
public SampleResponse create(SampleRequest request) {
    Sample sample = Sample.builder()...build();
    sampleMapper.insertSample(sample);
    return SampleResponse.from(sample);
}

// 수정
@Transactional
public SampleResponse update(String id, SampleRequest request) {
    Sample sample = sampleMapper.selectSampleById(id)
            .orElseThrow(() -> new SampleException(SampleErrorCode.NOT_FOUND));
    sample.update(...);
    sampleMapper.updateSample(sample);
    return SampleResponse.from(sample);
}
```

## 4. 페이징 처리

Mapper에서 `offset/limit` 파라미터와 `count` 쿼리를 분리하고, Service에서 `PageImpl`로 조합한다.

```java
// Mapper
List<Sample> selectAll(@Param("offset") long offset, @Param("limit") int limit);
long countAll();

// Service
@Transactional(readOnly = true)
public CommonPageResponse<SampleResponse> getList(Pageable pageable) {
    long totalCount = sampleMapper.countAll();
    List<Sample> content = sampleMapper.selectAll(pageable.getOffset(), pageable.getPageSize());
    return CommonPageResponse.from(new PageImpl<>(content, pageable, totalCount), SampleResponse::from);
}
```

## 5. 동기화 규칙 (핵심)

### 5-1. Mapper 메서드 변경 시

Mapper 인터페이스에 메서드를 **추가/변경/삭제**하면 다음을 반드시 함께 수행한다:

| 변경 대상 | Mapper XML |
|-----------|------------|
| 메서드 추가 | XML에 SQL 추가 |
| 메서드 변경 | XML SQL 수정 |
| 메서드 삭제 | XML에서 SQL 제거 |

### 5-2. Entity 필드 변경 시

Entity에 필드를 **추가/변경/삭제**하면 다음을 반드시 함께 수행한다:

| 변경 대상 | MyBatis 측 |
|-----------|------------|
| 필드 추가 | XML `<resultMap>`에 매핑 추가 + INSERT/UPDATE SQL에 컬럼 추가 |
| 필드명 변경 | XML `<resultMap>` property 수정 + SQL의 `#{}` 파라미터 수정 |
| 필드 삭제 | XML `<resultMap>`에서 제거 + INSERT/UPDATE SQL에서 컬럼 제거 |

### 5-3. 검색 조건(DTO) 변경 시

검색 조건 DTO에 필드를 추가/변경하면:

- **MyBatis**: XML의 `<sql>` 동적 조건(`<if>`) 반영

## 6. 동기화 체크리스트

새 도메인 또는 Mapper 메서드 작업 시 아래를 확인한다:

- [ ] Mapper 인터페이스의 모든 메서드에 대한 SQL이 XML에 존재하는가
- [ ] Mapper XML의 `<resultMap>`이 Entity의 모든 필드를 매핑하고 있는가
- [ ] INSERT SQL이 Entity의 모든 필드를 포함하는가
- [ ] UPDATE SQL이 변경 가능한 모든 필드를 포함하는가 (최초 등록 감사 필드 제외)
- [ ] Mapper XML의 `namespace`가 Mapper 인터페이스 FQCN과 일치하는가
- [ ] Service에서 신규 생성은 `insertXxx()`, 수정은 `updateXxx()`를 명시적으로 호출하는가
