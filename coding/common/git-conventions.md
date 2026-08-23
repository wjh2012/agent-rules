# Git 및 형상관리 규칙

## 1. 브랜치 전략 (Branch Strategy)
프로젝트는 단순화된 Git Flow(또는 GitLab Flow)를 기반으로 운영합니다.
- **`master`**: 프로덕션 환경에 배포 가능한 가장 안정적인 상태를 유지하는 브랜치입니다. PR 병합을 통해서만 반영합니다.
- **`develop`**: 다음 배포를 위해 기능들이 통합되는 기본 개발 브랜치입니다.
- **`feature/`**: 새로운 기능 개발 시 `develop`에서 분기합니다.
    - 명명 규칙: `feature/{이슈번호}-{기능명}` (예: `feature/12-add-email-login`)
- **`bugfix/`**: `develop` 브랜치의 버그를 수정할 때 사용합니다.
- **`hotfix/`**: 프로덕션(`master`)에서 발생한 긴급한 오류를 수정할 때 사용합니다.

## 2. 커밋 메시지 규칙 (Commit Message Convention)
커밋 메시지는 코드의 변경 이력을 쉽게 파악하고 자동화 도구와 연동하기 위해 [Conventional Commits](https://www.conventionalcommits.org/) 형식을 따릅니다.

## 3. 커밋 단위 규칙
가능하다면 실행 가능한 단위로 원자적으로 커밋한다.

**포맷:**
```text
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

**예시:**
```text
feat(auth): add email login functionality

- 이메일 및 비밀번호 유효성 검사 로직 추가
- JWT 발급 및 세션 관리 기능 구현
- 로그인 실패 시 에러 핸들링 로직 적용

Closes: #21
```