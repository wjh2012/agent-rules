---
name: file-io-tx
description: 파일 업/다운로드와 DB 트랜잭션 분리 규칙을 로드합니다. 파일 I/O가 @Transactional 밖에서 수행되도록 코드를 작성하거나 리뷰합니다.
---

아래 파일 I/O 트랜잭션 분리 규칙을 숙지하고, 이 규칙에 따라 파일 업/다운로드 관련 코드를 작성하거나 리뷰하세요.

## 파일 I/O 트랜잭션 분리 규칙

!`cat docs/rules/file-io-transaction.md`
