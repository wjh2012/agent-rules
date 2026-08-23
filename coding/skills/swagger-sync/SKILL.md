---
name: swagger-sync
description: SwaggerConfig의 GroupedOpenApi 그룹을 실제 Controller @RequestMapping과 동기화합니다. 누락 추가 + 고아 그룹 제거.
allowed-tools: Read, Grep, Glob, Edit
---

## SwaggerConfig 동기화

SwaggerConfig 파일과 모든 Controller의 @RequestMapping을 비교하여 **누락된 그룹은 추가**하고, **Controller가 삭제된 고아 그룹은 제거**합니다.

### 절차

1. **현재 SwaggerConfig 읽기**
   - `{api-module}/src/main/java/{base-package}/global/config/SwaggerConfig.java` 읽기
   - 기존 `GroupedOpenApi` 빈의 `pathsToMatch` 목록을 수집

2. **모든 Controller의 @RequestMapping 수집**
   - `{api-module}/src/main/java/**/api/*Controller.java` 파일을 모두 검색
   - 클래스 레벨 `@RequestMapping` 값을 추출

3. **양방향 비교**
   - **누락**: Controller는 있는데 SwaggerConfig에 없는 경로 → 추가 대상
   - **고아**: SwaggerConfig에는 있는데 매칭되는 Controller가 없는 경로 → 제거 대상

4. **그룹 추가** (누락이 있을 때)
   - 누락된 경로마다 새 `GroupedOpenApi` 빈을 추가
   - 그룹명: 기존 번호 체계를 이어서 `{번호}-{도메인명}` 형식
   - 메서드명: `{도메인}Api` (camelCase)
   - 경로: `/{base-path}/**`
   - `openAPI()` 빈 위에 삽입

5. **고아 그룹 제거** (고아가 있을 때)
   - 해당 `@Bean` 메서드 전체를 삭제
   - 제거 후 남은 그룹의 번호는 재정렬하지 않는다 (안정성 우선)

6. **결과 보고**

```
## swagger-sync 결과

### 추가
- {번호}-{도메인명}: /{path}/**

### 제거
- {번호}-{도메인명}: /{path}/** (Controller 없음)

### 변경 없음
동기화 완료, 변경 없음
```
