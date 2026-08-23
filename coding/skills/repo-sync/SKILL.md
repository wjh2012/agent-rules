---
name: repo-sync
description: JPA/MyBatis Repository 동기화를 검증하고 누락된 부분을 자동 생성합니다. Entity 필드 변경, Repository 메서드 추가/삭제 시 사용합니다.
---

JPA/MyBatis Repository 동기화 상태를 검증하고 누락된 부분을 보완하세요.

## 규칙
!`cat docs/rules/repository.md`

## 작업 절차

1. **대상 식별**: 인자가 있으면 해당 도메인, 없으면 아래 두 범위를 합산하여 Entity/Repository 변경을 감지
   - **커밋된 변경**: `git diff --name-only @{upstream}..HEAD` (upstream 미설정 시 `origin/{현재브랜치}..HEAD`)
   - **아직 커밋 안 한 변경**: `git diff --name-only` (unstaged) + `git diff --name-only --staged` (staged)
   - 두 결과를 합쳐서 Entity/Repository/Mapper 파일을 필터링
2. **현재 상태 수집**: 대상 도메인의 아래 파일을 모두 읽는다
   - `{Entity}.java` — 필드 목록
   - `{Entity}Repository.java` — 인터페이스 메서드
   - `Jpa{Entity}Repository.java` — Spring Data JPA 인터페이스
   - `Jpa{Entity}RepositoryAdapter.java` — JPA 어댑터
   - `{Entity}Mapper.java` — Mapper 인터페이스
   - `MyBatis{Entity}RepositoryAdapter.java` — MyBatis 어댑터
   - `{Entity}Mapper.xml` — Mapper XML
3. **동기화 검증**: 아래 항목을 점검
   - Repository 인터페이스의 모든 메서드가 JPA Adapter에 구현되었는가
   - Repository 인터페이스의 모든 메서드가 MyBatis Adapter에 구현되었는가
   - Mapper 인터페이스에 MyBatis Adapter가 호출하는 모든 메서드가 선언되었는가
   - Mapper XML에 Mapper 인터페이스의 모든 메서드에 대한 SQL이 존재하는가
   - Mapper XML `<resultMap>`이 Entity의 모든 필드를 매핑하는가 (BaseEntity 공통 필드 포함)
   - INSERT SQL이 Entity의 모든 필드를 포함하는가
   - UPDATE SQL이 변경 가능한 모든 필드를 포함하는가 (최초 등록 감사 필드 제외)
   - Mapper XML의 `namespace`가 Mapper 인터페이스 FQCN과 일치하는가
4. **누락 보완**: 누락된 파일/메서드/SQL/매핑을 자동 생성
5. **결과 보고**

## 자동 생성 규칙

### Mapper XML 컬럼 매핑
- Entity 필드명 → DB 컬럼명: camelCase → UPPER_SNAKE_CASE
- 예: `categoryId` → `CATEGORY_ID`, `createdAt` → `CREATED_AT`

### BaseEntity 공통 필드 (모든 resultMap에 포함)
```xml
<result property="createdAt" column="CREATED_AT"/>
<result property="createdBy" column="CREATED_BY"/>
<result property="updatedAt" column="UPDATED_AT"/>
<result property="updatedBy" column="UPDATED_BY"/>
```

### MyBatis Adapter save() 패턴
```java
@Override
public {Entity} save({Entity} entity) {
    if (entity.isNew()) {
        {entity}Mapper.insert{Entity}(entity);
    } else {
        {entity}Mapper.update{Entity}(entity);
    }
    return entity;
}
```

## 출력 형식

```
## 동기화 결과: {도메인명}

### ✅ 동기화 완료
- [항목] 설명

### 🔧 자동 보완됨
- [항목] `파일` — 추가/수정 내용

### ❌ 수동 확인 필요
- [항목] `파일` — 사유
```
