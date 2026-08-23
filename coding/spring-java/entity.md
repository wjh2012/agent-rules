# Entity 규칙 — Java · Spring

> `BaseEntity` 등 공통 상위 클래스명은 프로젝트 `CLAUDE.md` 에서 바인딩한다.
> ORM 별 차이는 본문에 표시했다. MyBatis 는 [mybatis/repository.md](mybatis/repository.md) 도 함께 본다.

데이터 무결성과 객체지향적 설계를 위해 다음 규칙을 엄수합니다.

## 1. Entity 상태 변경은 의도(Intent) 기반 비즈니스 메서드로만

Entity 클래스의 상태 변경은 **의도가 드러나는 비즈니스 메서드**를 통해서만 수행합니다. `@Setter` 대신 명시적인 메서드를 사용하며, 하나의 `update()` 메서드에 모든 필드를 몰아넣지 않고 **변경 의도별로 메서드를 분리**합니다.

❌ **Bad — @Setter 사용**:

```java
@Entity
@Setter // 의도가 불명확
public class Order {
    private OrderStatus status;
}
```

❌ **Bad — 단일 update()에 모든 필드**:

```java
// 코드만 바꾸고 싶어도 명칭·설명을 함께 넘겨야 함
public void update(String code, String name, String desc) { ... }
```

✅ **Good — 의도별 메서드 분리**:

```java
/** 항목 코드를 변경한다. */
public void renameCode(String code) {
    Assert.hasText(code, "항목 코드는 필수입니다.");
    this.code = code;
}

/** 항목명을 변경한다. */
public void renameName(String name) {
    Assert.hasText(name, "항목명은 필수입니다.");
    this.name = name;
}

/** 설명을 갱신한다. */
public void updateDescription(String description) {
    this.description = description;
}

/** 활성화 상태를 변경한다. */
public void changeActiveStatus(String activeYn) {
    this.activeYn = activeYn;
}
```

### 메서드 네이밍 가이드

| 의도 | 네이밍 패턴 | 예시 |
|------|------------|------|
| 식별 코드 변경 | `renameXxx()` | `renameCode()` |
| 명칭 변경 | `renameName()`, `renameTitle()` | `renameName()` |
| 상태 전이 | `activate()`, `deactivate()`, `close()` | `deactivate()` |
| 설명·메모 갱신 | `updateDescription()`, `updateMemo()` | `updateDescription()` |
| 복합 속성 갱신 | `updateXxxInfo()` | `updateConnectionInfo(host, port)` |

## 2. Entity 클래스 어노테이션 표준

모든 Entity 클래스는 다음 어노테이션 조합을 표준으로 사용합니다.

```java
@Entity
@Table(name = "TB_XXX")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
```

- `@NoArgsConstructor(access = AccessLevel.PROTECTED)`: JPA 스펙상 기본 생성자가 필요하나, 외부 직접 호출을 차단합니다.

**`@Builder`는 생성자 레벨에만 적용합니다.** 필수 필드만 선별하여 빌더를 생성합니다.

- 클래스 레벨 `@Builder` + `@AllArgsConstructor`는 **사용하지 않습니다.** 모든 필드가 빌더에 노출되어 의도하지 않은 필드 설정이 가능해지기 때문입니다.
- 기본값이 필요한 필드는 **생성자 내부**에서 직접 설정합니다. (예: `this.delYn = "N"`, `this.id = generateId()`)
- `@Builder.Default`는 클래스 레벨 `@Builder` 전용이므로 사용하지 않습니다.

## 3. 생성자(빌더) 및 팩토리 메서드

- 객체 생성은 **정적 팩토리 메서드**를 우선적으로 고려합니다. (예: `of()`, `from()`)
- 생성자가 복잡할 경우, 특정 생성자 상단에 `@Builder`를 적용하여 필수 필드 입력을 명시화합니다.

### 빌더 생성자 작성 규칙

1. **`id` 파라미터는 선택적으로 수용한다.** 외부에서 ID가 전달되면 그대로 사용하고, `null`이면 자동 생성한다. (테스트·마이그레이션 시 ID 지정 가능)
2. **필수 파라미터는 `Assert.hasText()`로 검증한다.** 빈 문자열/null이 들어오면 즉시 실패시킨다.
3. 기본값이 필요한 필드는 생성자 내부에서 직접 설정한다.

```java
/**
 * 항목 생성을 위한 빌더.
 *
 * @param id          고유 식별자 (선택, 미지정 시 자동 생성)
 * @param categoryId  소속 카테고리 ID (필수)
 * @param code        항목 코드 (필수, 중복 불가)
 * @param name        항목명 (필수)
 * @param description 설명 (선택)
 * @throws IllegalArgumentException 필수 파라미터가 누락되거나 빈 문자열인 경우 발생
 */
@Builder
private Item(String id, String categoryId, String code, String name,
             String description) {
    Assert.hasText(categoryId, "카테고리 ID는 필수입니다.");
    Assert.hasText(code, "항목 코드는 필수입니다.");
    Assert.hasText(name, "항목명은 필수입니다.");

    this.id = (id != null) ? id : UuidGenerator.generate();
    this.categoryId = categoryId;
    this.code = code;
    this.name = name;
    this.description = description;
}
```

## 4. 연관관계 설계 규칙

- ID 기반 연관관계: Entity 간 연관관계는 ID 필드를 통한 간접 참조로 구현한다. (@ManyToOne, @OneToMany 대신 외래 키 ID 필드를 직접 정의)
- 이유: 객체 간의 결합도를 낮추고, MyBatis 사용 시 SQL 매핑의 명확성을 확보하며, N+1 문제를 방지하기 위함입니다.
- 데이터 조회: 연관된 데이터가 필요한 경우, Mapper(XML)에서 JOIN을 통해 한 번에 조회하여 DTO로 반환하거나, 각각의 Mapper를 통해 필요한 시점에 조회합니다.

## 5. 공통 필드 상속 (BaseEntity)

모든 Entity는 `BaseEntity<ID>`를 상속받아 감사(Audit) 필드와 `isNew` 판별을 공통 관리합니다.

### 공통 필드

| 필드        | 컬럼명       | 타입              | 설명                          |
|-------------|-------------|-------------------|-------------------------------|
| `createdAt` | `CREATED_AT` | `LocalDateTime`   | 최초 등록일시 (`updatable = false`) |
| `createdBy` | `CREATED_BY` | `String(100)`     | 최초 등록자 ID (`updatable = false`) |
| `updatedAt` | `UPDATED_AT` | `LocalDateTime`   | 최종 변경일시                   |
| `updatedBy` | `UPDATED_BY` | `String(100)`     | 최종 변경자 ID                  |

위 필드들은 Audit Interceptor에 의해 자동으로 설정되므로 비즈니스 로직에서 수동 변경하지 않는다.

> **[JPA 전용]** `AuditingEntityListener` 설명은 JPA 환경 전용이다. MyBatis 환경에서는 `AuditingEntityListener`가 동작하지 않으며, MyBatis Audit Interceptor가 해당 역할을 대신한다.

### isNew 판별 메커니즘 *(JPA 전용 — MyBatis에서는 동작하지 않음)*

> **[JPA 전용]** 아래 `isNew`/`Persistable` 흐름은 JPA 환경 전용이다. MyBatis 환경에서는 `isNew()`, `@PostPersist`, `@PostLoad` 콜백이 실제로 호출되지 않는다. `BaseEntity`에 관련 코드가 남아 있는 것은 **필드-동작 문서화 목적**이다.

`BaseEntity`는 `Persistable<ID>`를 구현하여 신규/기존 엔티티를 판별합니다.

- **신규(`isNew = true`)** → `persist()` 실행 (INSERT)
- **기존(`isNew = false`)** → `merge()` 실행 (SELECT 후 UPDATE)

ID를 미리 할당하는 전략(UUID 등)에서는 ID가 이미 존재하므로 JPA가 기존 엔티티로 오인하여 불필요한 SELECT가 발생합니다. `Persistable`을 구현하면 `isNew` 필드로 직접 판별하여 이 문제를 해결합니다.

```
[JPA 흐름 — 참조용]
1. 엔티티 생성 → isNew = true (필드 기본값)
2. save() 호출 → isNew()가 true 반환 → persist() 실행 (INSERT)
3. @PostPersist 콜백 → isNew = false
4. 이후 조회 시 → @PostLoad 콜백 → isNew = false
```

### MyBatis 환경 주의사항

JPA를 제거하고 **MyBatis 단일화** 구성을 사용하는 경우 아래 사항에 유의한다.

| 항목 | JPA 환경 | MyBatis 환경 |
|------|----------|---------------------|
| 감사 필드 자동 설정 | `AuditingEntityListener` | MyBatis Audit Interceptor |
| `isNew()` 판별 | `Persistable` 구현으로 동작 | **동작하지 않음** |
| `@PostPersist` 콜백 | save() 직후 호출 | **호출되지 않음** |
| `@PostLoad` 콜백 | 조회 직후 호출 | **호출되지 않음** |

- `BaseEntity`의 `@Entity`, `@Table`, `@Column`, `Persistable`, `@PostPersist`, `@PostLoad` 선언은 **필드-컬럼 매핑 문서화 목적**으로 유지한다. 실제 영속성 동작은 MyBatis Mapper XML이 담당한다.
- INSERT/UPDATE 시 감사 필드(`createdAt`, `createdBy`, `updatedAt`, `updatedBy`) 설정은 Audit Interceptor에서 처리하므로 비즈니스 로직에서 직접 설정하지 않는다.

### getId() 오버라이드 필수

각 엔티티의 PK 필드명과 타입이 다르므로 하위 엔티티에서 반드시 `getId()`를 직접 구현해야 한다.

```java
@Entity
@Table(name = "TB_ORDER")
@Getter
public class Order extends BaseEntity<String> {

    @Id
    @Column(name = "ORDER_ID", length = 36)
    private String orderId;

    @Override
    public String getId() {
        return this.orderId;
    }

    // ... existing code ...
}
```

> **주의**: `getId()`를 구현하지 않으면 컴파일 에러가 발생한다.

## 6. UUID PK 생성은 공통 유틸리티로 통일

모든 엔티티의 UUID PK 생성은 프로젝트 공통 UUID 생성 유틸리티를 사용한다.

- Hibernate `@UuidGenerator`, `@GeneratedValue` 사용 **금지** — 생성 전략을 애플리케이션 레벨에서 통일 관리한다.
- `UUID.randomUUID()` — 엔티티 PK에 **사용 금지**. 임시 파일명 등 비영속 용도에서만 허용.
- UUID 생성기 직접 호출 **금지** — 반드시 공통 유틸리티를 경유한다.
- PK 할당 시점: **빌더 생성자** 또는 **팩토리 메서드** 내부에서 수행한다.

```java
@Builder
private MyEntity(String name) {
    this.id = UuidGenerator.generate();
    this.name = name;
}
```

## 7. 초기화는 필드 초기화 또는 팩토리 메서드로

- 초기화 로직은 `@PrePersist` 대신 필드 초기화 또는 팩토리 메서드에서 명시적으로 처리합니다. (가독성·명시성 확보)
- **대안**: 기본값은 **필드 초기화**(예: `private String delYn = "N"`)로 설정하고, PK 생성·시각 설정 등은 **팩토리 메서드** 또는 **빌더**에서 명시적으로 처리합니다.

## 8. Surrogate ID + 테이블 네이밍 컨벤션

모든 엔티티는 **surrogate ID**(대체키)를 PK로 사용한다. 비즈니스 의미가 있는 natural key는 unique 제약 컬럼으로 관리한다.

| 항목 | 규칙 |
|------|------|
| 테이블명 | `TB_` 접두어 필수 (예: `TB_USER`, `TB_ORDER`) |
| PK 필드명 | `private String id` (모든 엔티티 통일) |
| PK 컬럼명 | `{TABLE_NAME(TB_ 제외)}_ID` (단수형, 예: `USER_ID`, `ORDER_ID`) |
| PK 생성 | 공통 UUID 생성 유틸리티 — 빌더/팩토리 메서드 내부에서 할당 |
| FK 컬럼명 | `{PARENT_TABLE(TB_ 제외)}_ID` (예: `CATEGORY_ID`) |
| Natural Key | unique 제약으로 관리 (예: `USER_CODE`, `ITEM_CD`) |
| BaseEntity | 감사 필드만 관리. `@Id`는 각 엔티티에서 개별 선언 |
| 감사 필드 | `CREATED_BY`, `UPDATED_BY`에는 사용자 식별자 저장 |
| ID `@Column` | `updatable = false` 필수 |

```java
@Entity
@Table(name = "TB_ITEM")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Item extends BaseEntity {

    /** 항목의 고유 식별자.
     * 공통 UUID 생성 유틸리티를 통해 생성된 UUID(36자)를 사용하며, 생성 후 변경할 수 없다.
     */
    @Id
    @Column(name = "ITEM_ID", length = 36, updatable = false)
    private String id;

    /** 소속 카테고리 ID (FK, {@link Category#getId()}) */
    @Column(name = "CATEGORY_ID", length = 36)
    private String categoryId;

    /** 항목 코드. 시스템 내에서 유니크해야 함. */
    @Column(name = "ITEM_CD", unique = true, nullable = false)
    private String code;

    @Builder
    private Item(String id, String categoryId, String code, String name,
                 String description) {
        Assert.hasText(categoryId, "카테고리 ID는 필수입니다.");
        Assert.hasText(code, "항목 코드는 필수입니다.");
        Assert.hasText(name, "항목명은 필수입니다.");

        this.id = (id != null) ? id : UuidGenerator.generate();
        this.categoryId = categoryId;
        this.code = code;
        // ...
    }
}
```

## 9. Entity Javadoc 규칙

모든 Entity 클래스는 아래 Javadoc 규칙을 따른다.

### 클래스 Javadoc

- 엔티티의 역할을 한 문장으로 기술
- 부가 설명이 있으면 `<p>` 태그로 단락 분리
- 관련 엔티티가 있으면 `@see`로 참조 링크 제공

```java
/**
 * 항목 정보를 관리하는 엔티티.
 * <p>
 * {@link Category}에 속하며, 개별 항목의 코드·명칭·설명·활성 상태를 관리한다.
 *
 * @see Category
 */
```

### 필드 Javadoc

- 한 줄 `/** ... */` 형태로 필드의 의미·용도를 기술
- 예시 값이나 제약사항이 있으면 괄호로 병기
- **ID 필드**: UUID 생성 전략과 불변성을 명시

```java
/** 항목의 고유 식별자.
 * 공통 UUID 생성 유틸리티를 통해 생성된 UUID(36자)를 사용하며, 생성 후 변경할 수 없다.
 */
@Id
@Column(name = "ITEM_ID", length = 36, updatable = false)
private String id;
```

- **FK 필드**: `(FK, {@link Parent#getId()})` 형태로 참조 엔티티 명시

```java
/** 소속 카테고리 ID (FK, {@link Category#getId()}) */
@Column(name = "CATEGORY_ID", length = 36)
private String categoryId;
```

- **일반 필드**: 의미·용도를 간결하게 기술

```java
/** 항목 코드. 시스템 내에서 유니크해야 함. */
@Column(name = "ITEM_CD", unique = true, nullable = false)
private String code;

/** 항목명 (화면 노출용) */
@Column(name = "ITEM_NM", nullable = false)
private String name;

/** 활성화 여부 (Y/N) */
@Column(name = "ACTIVE_YN")
private String activeYn;
```

### 생성자(빌더) Javadoc

- 생성 목적을 기술하고, `@param`으로 각 파라미터 설명 (필수/선택 명시)
- `id` 파라미터는 **"선택, 미지정 시 자동 생성"**으로 명시
- `@throws`로 검증 실패 시 예외 명시

```java
/**
 * 항목 생성을 위한 빌더.
 *
 * @param id          고유 식별자 (선택, 미지정 시 자동 생성)
 * @param categoryId  소속 카테고리 ID (필수)
 * @param code        항목 코드 (필수, 중복 불가)
 * @param name        항목명 (필수)
 * @param description 설명 (선택)
 * @throws IllegalArgumentException 필수 파라미터가 누락되거나 빈 문자열인 경우 발생
 */
```

### 비즈니스 메서드 Javadoc

- 메서드의 의도를 한 문장으로 기술
- `@param`, `@throws` 태그 포함

```java
/**
 * 항목 코드를 변경한다.
 *
 * @param code 새로운 항목 코드 (필수, 중복 불가)
 * @throws IllegalArgumentException 항목 코드가 빈 문자열인 경우 발생
 */
```
