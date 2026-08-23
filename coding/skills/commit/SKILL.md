---
name: commit
description: Git 커밋 시 Conventional Commits 규칙을 적용하여 커밋 메시지를 생성하고 커밋합니다.
---

아래 Git 규칙을 숙지하고, 변경 사항을 분석하여 커밋하세요.

## Git 규칙

!`cat docs/rules/git-conventions.md`

## 작업 절차

1. `git diff --staged`로 스테이징된 변경 확인. 없으면 `git diff`로 전체 변경 확인.
2. 변경 사항을 분석하여 Conventional Commits 형식의 커밋 메시지 작성.
3. 관련 파일을 스테이징하고 커밋 실행.
4. 커밋 메시지 끝에 `Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>` 추가.
