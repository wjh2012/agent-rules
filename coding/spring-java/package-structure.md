# 패키지 구조 규칙 — Java · Spring

> `{project}` · `{group}` · `{도메인}` 은 프로젝트에서 치환한다.

## 멀티 모듈 구성

| 모듈 | Base Package | 역할 |
|------|--------------|------|
| `{project}-api` | `{group}.{project}api` | 단일 실행 모듈 (bootJar), 업무 도메인 통합 |
| `{project}-core` | `{group}.{project}core` | 공통 라이브러리 (공유 도메인, 자체 global 인프라) |
| `{project}-worker` | `{group}.{project}worker` | 워커 실행 모듈 (독립 bootJar) |

### Gradle 의존 관계

```
{project}-api (단일 실행 모듈)
  └→ {project}-core

{project}-worker
  └→ {project}-core
```

---

---

## API 모듈 패키지 구조

단일 실행 모듈 — 모든 업무 도메인 통합.

```
{group}.{project}api
├── domain/
│   ├── {도메인A}/               # 업무 도메인 (예: user, order, product 등)
│   ├── {도메인B}/
│   └── ...
│       └── {domain}/
│           ├── api/
│           ├── dto/
│           ├── entity/
│           ├── exception/
│           ├── repository/     # Mapper 인터페이스
│           └── service/
├── global/
│   ├── config/                 # 공통 설정 (Spring Security, MyBatis, Swagger 등)
│   ├── entity/                 # BaseEntity
│   ├── exception/              # BusinessException, ErrorCode, GlobalExceptionHandler 등
│   ├── response/               # 공통 응답 래퍼
│   ├── util/                   # 공통 유틸
│   └── storage/                # 공용 스토리지
```

**빈 스캔 범위:**
- `@SpringBootApplication(scanBasePackages)`: API 모듈 + Core 모듈 패키지
- `@MapperScan`: 동일 범위

---

## 공통 규칙

### 1. 도메인 내부 계층

모든 도메인은 다음 하위 패키지를 가진다 (필요한 것만 생성):

| 패키지        | 필수 | 역할                                                    |
|--------------|------|--------------------------------------------------------|
| `api`        | O    | Controller                                             |
| `service`    | O    | 비즈니스 로직                                           |
| `repository` | O    | Mapper 인터페이스 직접 위치 |
| `entity`     | O    | Entity                                                 |
| `dto`        | O    | 요청/응답 DTO                                          |
| `exception`  | -    | 도메인 전용 예외                                        |
| `util`       | -    | 도메인 전용 유틸리티                                    |

### 2. 의존성 방향 (단방향)

```
api → service → repository → entity
         ↓
        dto
```

- 역방향 의존 금지.
- `global` 패키지는 모든 도메인에서 참조 가능하지만, `global`이 특정 도메인을 참조하면 안 된다.
- 도메인 간 의존: 같은 계층의 `service`를 주입하여 사용. `repository`를 직접 참조하지 않는다.
- 모듈 간 의존: Gradle 의존 방향을 따른다. 역방향 금지.

### 3. MyBatis Mapper XML 위치

```
src/main/resources/mapper/{도메인명}/  # 도메인별 디렉토리
```

모듈이 여러 개인 경우 충돌 방지를 위해 `classpath*:mapper/**/*.xml` 패턴 사용.

### 4. 새 도메인 생성 시 체크리스트

1. 어느 모듈에 속하는지 결정 (`{project}-api` vs `{project}-core`)
2. 필수 패키지 (`api`, `service`, `repository`, `entity`, `dto`) 생성
3. `repository/` 아래에 Mapper 인터페이스 직접 배치
4. MyBatis Mapper XML 디렉토리 생성: `resources/mapper/{도메인명}/`
5. 도메인 전용 예외가 필요하면 `exception/` 패키지 추가
