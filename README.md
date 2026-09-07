# 에이전트 하네스 가이드

React 프론트엔드와 Django 백엔드가 함께 있는 저장소에서 코딩 에이전트가 작업 범위를 빠르게 파악하고, 필요한 지침만 빠짐없이 읽도록 만든 작업 지침 모음이다.

이 저장소는 특정 프로젝트의 코드베이스에서 사용하는 하네스 문서만 추출한 비공개 배포본이다. 예제의 도메인 용어와 경로는 원본 코드와의 일치를 위해 유지한다.

## 대상 코드베이스

| 영역 | 기준 스택 | 문서가 다루는 범위 |
| --- | --- | --- |
| Client | React 18, TypeScript 5, Vite, React Router, Chakra UI | 컴포넌트·hook·상태·접근성·성능 |
| Client data | TanStack Query, React Hook Form, Zod | 서버 상태·폼·입력 검증 |
| Client test | Vitest, Testing Library, user-event, MSW | UI·hook·API 응답 Mock·Router 테스트 |
| API contract | Django Ninja OpenAPI, Orval, openapi-typescript-codegen | 타입·Zod·Mock 생성물 동기화 |
| Server | Python, Django 4.2, Django Ninja, Pydantic | 모델·스키마·서비스·권한·트랜잭션 |
| Server test | pytest, pytest-django, factory_boy | fixture·Factory·DB·엔드포인트·모델 테스트 |
| Runtime | PostgreSQL, Celery, Docker/k8s 기반 개발 환경 | 커밋 경계·비동기 부수효과·테스트 실행 |

## 예상 디렉터리 구조

```text
repository/
├── AGENTS.md
├── client/
│   ├── AGENTS.md
│   └── guides/
└── server/
    ├── AGENTS.md
    ├── main/AGENTS.md
    └── guides/
```

## 읽는 순서

1. 저장소 루트의 `AGENTS.md`를 읽는다.
2. 변경 위치가 `client/`면 `client/AGENTS.md`, `server/`면 `server/AGENTS.md`를 읽는다.
3. 작업 동작과 맞는 가이드만 선택해 해당 파일 전체를 읽는다.
4. 여러 조건에 해당하면 가이드의 합집합을 읽고, 문서에 명시된 경우에만 연결된 가이드를 추가한다.
5. 예제 링크는 패턴을 확인할 때 필요한 심볼과 주변 코드만 확인한다. 연결된 소스 파일 전체를 미리 읽지 않는다.

파일명만 보고 문서를 고르지 않는다. 변경하는 동작을 기준으로 고른다. 같은 작업에서 이미 읽은 문서는 내용이 바뀌지 않았다면 다시 읽지 않는다.

## 가이드 구성

### Client

- `react-frontend.md`: React 컴포넌트·hook·상태·API 연동
- `api-schema-generation.md`: OpenAPI에서 타입·Zod·Mock 생성물 갱신
- `testing-patterns.md`: Vitest와 Testing Library 테스트 패턴
- `mock-data-guide.md`: 생성 Mock과 관계형 테스트 데이터

### Server

- `models.md`: 모델·관계·제약·QuerySet
- `schema.md`: Django Ninja 입력·출력 스키마
- `service-layer.md`: API·권한·업무 검증·트랜잭션
- `factory.md`: factory_boy Factory
- `fixture.md`: pytest fixture 탐색과 작성
- `testing.md`: 테스트 공통 규칙
- `testing-endpoint.md`: HTTP 엔드포인트 테스트
- `testing-models.md`: 모델 제약·메서드·`full_clean()` 테스트
- `testing-db.md`: `db`, `transactional_db`, `on_commit`, atomic 경계

## 적용할 때 확인할 것

- `AGENTS.md`는 저장소 루트와 각 앱 경로에 복사한다.
- 가이드의 상대 경로 예시는 `client/src`, `server/main`, `server/main/_test_fixture` 같은 원본 구조를 전제로 한다.
- OpenAPI·Mock·Orval 생성물은 직접 편집하지 않고 원본 설정이나 서버 스키마를 수정한 뒤 다시 생성한다.
- 서버 문서의 테스트 명령은 컨테이너/k8s 실행 스크립트를 전제로 한다. 로컬 실행 환경에서는 같은 테스트 대상을 프로젝트 명령에 맞게 바꾼다.
- 이 문서는 코딩 에이전트의 작업 범위와 읽기 순서를 정하는 하네스다. 제품 사용 설명서나 일반 Django 입문서는 포함하지 않는다.

## 원본과의 관계

이 패키지는 `/client/guides`, `/server/guides`와 각 경로의 에이전트 진입점을 그대로 보관하기 위한 것이다. 대상 저장소의 소스 코드, 배포 설정, 비밀값, 테스트 데이터는 포함하지 않는다.
