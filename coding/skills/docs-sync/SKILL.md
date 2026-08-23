---
name: docs-sync
description: 코드 변경 후 docs/rules/ 문서 동기화가 필요한지 판단하고 갱신합니다. Entity, API, 파일 처리, 스토리지 등의 코드 변경 시 사용합니다.
---

코드 변경사항을 분석하고 docs/rules/ 문서를 동기화하세요.

## 작업 절차

1. 아래 두 범위를 합산하여 변경된 코드 파일 확인 (인자가 있으면 해당 범위 사용)
   - **커밋된 변경**: `git diff --name-only @{upstream}..HEAD` (upstream 미설정 시 `origin/{현재브랜치}..HEAD`)
   - **아직 커밋 안 한 변경**: `git diff --name-only` (unstaged) + `git diff --name-only --staged` (staged)
2. 변경된 코드 파일을 읽고 내용 분석
3. 변경 내용이 영향을 주는 docs/rules/ 문서 식별
4. 해당 문서를 읽고 코드와 불일치하는 부분 확인
5. 문서 내용을 코드에 맞게 갱신
6. 변경 없으면 "동기화 불필요" 보고

## 문서 매핑

| 코드 변경 영역 | 대상 문서 |
|---|---|
| Entity 필드/어노테이션 | docs/rules/entity.md |
| Controller/DTO/API | docs/rules/api.md |
| 파일 메타데이터 | docs/guides/file-metadata-guide.md |
| 스토리지 (FileStorageService/RemoteStorageService) | docs/guides/common-storage-guide.md |
| 벤치마크/임시파일 | docs/guides/storage-benchmark-guide.md, docs/guides/temp-file-policy.md |
| 파일 I/O + 트랜잭션 | docs/rules/file-io-transaction.md |
| 패키지 구조 | docs/rules/package-structure.md |
| Repository/Mapper | docs/rules/repository.md |

## 갱신 판단 기준

**필요**: 필드/타입 변경, API 추가/삭제, 설정값 변경, 패키지 구조 변경, 새 패턴 도입
**불필요**: 내부 리팩토링, 버그 수정, 테스트만 변경, 포맷팅만 변경

## 규칙

- 코드에 실제로 반영된 패턴만 문서에 기록
- 기존 문서 구조/형식 유지
- 불필요한 내용 추가 금지
- 확실하지 않은 내용은 코드를 직접 읽어서 확인

## 출력 형식

```
## 동기화 결과

### 갱신됨
- `docs/rules/xxx.md` — 변경 내용 요약

### 동기화 불필요
- `docs/rules/yyy.md` — 사유
```
