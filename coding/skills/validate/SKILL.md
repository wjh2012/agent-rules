---
name: validate
description: 작성한 코드가 docs/rules 규칙에 맞는지 검증합니다. Entity, API, 파일 처리, 공통 코딩 규칙을 종합적으로 점검합니다.
---

변경된 코드 또는 지정된 파일을 docs/rules 규칙에 따라 검증하세요.

## 검증 대상 규칙

아래 규칙 문서들을 모두 읽고, 해당하는 규칙만 선별하여 검증하세요.

### Entity 규칙
!`cat docs/rules/entity.md`

### API 규칙
!`cat docs/rules/api.md`

### 파일 메타데이터 규칙
!`cat docs/guides/file-metadata-guide.md`

### 공용 스토리지 규칙
!`cat docs/guides/common-storage-guide.md`

### 임시 파일 정책
!`cat docs/guides/temp-file-policy.md`

### 파일 I/O 트랜잭션 분리 규칙
!`cat docs/rules/file-io-transaction.md`

### 패키지 구조 규칙
!`cat docs/rules/package-structure.md`

### Repository 규칙 (JPA/MyBatis 동기화)
!`cat docs/rules/repository.md`

## 검증 절차

1. **대상 파일 식별**: 인자가 있으면 해당 파일, 없으면 `git diff --name-only HEAD`로 변경된 `.java` 파일을 대상으로 한다.
2. **파일 읽기**: 대상 파일들을 모두 읽는다.
3. **규칙 매칭**: 각 파일의 유형(Entity, Controller, DTO, Service, Repository)에 따라 해당하는 규칙을 선별한다.
4. **위반 사항 검출**: 규칙 위반을 찾아 구체적인 위치(파일:라인)와 함께 보고한다.
5. **결과 보고**: 아래 형식으로 출력한다.

## 출력 형식

```
## 검증 결과

### ✅ 통과 항목
- [규칙명] 설명

### ❌ 위반 항목
- [규칙명] `파일:라인` — 위반 내용 및 수정 방법

### ⚠️ 권고 사항
- [규칙명] 설명
```
