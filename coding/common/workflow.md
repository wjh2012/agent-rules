# 워크플로우 규칙

## Skill Auto-Detection

- 사용자 요청 내용에 맞는 프로젝트 스킬(`.claude/skills/`)이 있으면 자동으로 감지하여 Skill tool로 호출한다.
- **어떤 요청에 어떤 스킬을 쓰는지는 프로젝트 `CLAUDE.md` 에 매핑을 적어둔다.**
  스킬을 설치만 하고 매핑을 안 적으면 호출되지 않는다.

## Post-Implementation Validation

- 코드 작성/수정 작업 완료 후, 반드시 `validate` 스킬을 실행하여 규칙 위반 여부를 점검한다.
- 위반 사항이 발견되면 즉시 수정하고, 수정 결과를 사용자에게 공유한다.

## Commit Rule

- Git 커밋을 수행할 때 반드시 `/commit` 스킬을 사용한다. 직접 `git commit` 명령을 실행하지 않는다.
- Conventional Commits 형식 준수. (상세: git-conventions.md)

## Plan Review

- `/plan` 모드에서 구현 계획 작성 완료 후, 반드시 `plan-review` 에이전트를 실행하여 과도한 설계, 누락된 고려사항, 규칙 위반을 점검한다.
- 리뷰 결과가 **OK**가 아닌 경우, 지적 사항을 반영하여 계획을 수정하고 다시 `plan-review`를 실행한다. 모든 항목이 OK일 때까지 이 과정을 반복한다.
- 리뷰에서 지적이 발생하면 **plan-review의 지적 사항 → 그에 따른 계획 변경 내용**을 사용자에게 공유하고, 다음 리뷰 사이클 전에 피드백을 받는다.
- 요구사항이 모호하거나 설계 방향이 여러 갈래인 경우, 임의로 판단하지 말고 사용자에게 선택지를 제시하여 확인을 받는다.

## Changelog

- 대규모 변경(아키텍처 변경, 모듈 추가/삭제, 핵심 로직 재설계 등)이 발생하면 `docs/changelog/`에 변경 이력 문서를 작성한다.
- 파일명: `{변경-주제}.md` (예: `convert-engine-migration.md`)
- 내용: 변경 배경, 주요 변경 사항, 영향 범위를 포함한다.

## Action Restrictions

- External API Calls: Only call external APIs when explicitly requested.
- Build/Test: Only run build or test commands when explicitly requested.
- Reformat/Lint: Only reformat code or run linters when explicitly requested.
- Partial Code: Use // ... existing code ... to represent unchanged parts.
