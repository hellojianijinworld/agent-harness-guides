# Server 코딩 지침

파일명은 탐색 힌트다. 변경·검토하는 동작에 해당하는 가이드를 각각 전체로 읽는다. 기존 코드에서 로직을 옮기는 작업도 적용 대상이다.

| 작업 | 읽을 가이드 |
| --- | --- |
| 모델·관계·제약·조회 프로퍼티·QuerySet 변경 (`models.py`, `querysets.py`, `mixins.py` 등) | [모델](guides/models.md) |
| 요청·응답 타입, 직렬화, 입력 변환 변경 (`schema.py`, `schema_*.py`, `common_schema.py` 등) | [스키마](guides/schema.md) |
| API·권한·비즈니스 검증·데이터 변경·트랜잭션 (`api*.py`, `services.py`, `services/**/*.py`, task 등 위치 무관) | [서비스와 API](guides/service-layer.md) |
| 테스트 작성·수정 | [테스트 공통](guides/testing.md) |
| HTTP 엔드포인트 테스트 | 테스트 공통 + [엔드포인트 테스트](guides/testing-endpoint.md) |
| 모델 제약·메서드·`full_clean` 테스트 | 테스트 공통 + [모델 테스트](guides/testing-models.md) |
| `on_commit`·atomic 예외 이후 조회·실제 커밋·동시성 테스트 | 테스트 공통 + [DB fixture](guides/testing-db.md) |
| Factory 정의·수정 (`factories.py` 등) | [Factory 작성](guides/factory.md) |
| fixture 탐색·정의·재정의 (`conftest.py`, `_test_fixture/` 등) | [Fixture](guides/fixture.md) |

테스트만 수정하면 구현 가이드를 추가로 읽을 필요는 없다. 기존 Factory 호출에 값만 넘기는 테스트는 Factory 작성 가이드가 필요 없다. fixture 탐색도 사용법을 이미 아는 기존 fixture 호출과 구분한다.
서버 endpoint·요청·응답 계약을 변경하면 [클라이언트 지침](../client/AGENTS.md)과 [API 스키마 생성](../client/guides/api-schema-generation.md)을 읽고 생성물 반영 범위를 확인한다.

## 공통

- Django Ninja 라우터와 기존 Django 인증을 사용한다. 엔드포인트마다 접근 권한을 확인하고, 사용자 입력은 스키마·서비스 경계에서 검증한다.
- 시크릿·개인정보는 로그에 출력하지 않는다.
- `# ─── 섹션명 ───` 같은 구역 나누기 주석을 작성하지 않는다. 클래스와 빈 줄로 구분한다.

## 검증과 관리 명령

아래 명령은 프로젝트 루트에서 실행한다. 테스트는 k8s 파드에서 실행하며 테스트 경로는 `server/main/` 기준이다.
서버 코드 수정 후 `manage.py check`와 변경 범위의 테스트를 실행한다. 문서만 변경했다면 링크·예제·실제 설정과의 일치를 확인한다.

```bash
scripts/run-k8s/main.sh python manage.py check
scripts/run-k8s/main-pytest.sh apps/<앱>/tests/<테스트 파일>.py -v
```

앱·마이그레이션 파일 생성은 직접 작성하지 않고 관리 명령을 사용한다. 마이그레이션은 전용 스크립트로 생성하고 파드에서 로컬로 복사된 결과를 확인한다.

```bash
scripts/run-k8s/main.sh python manage.py startapp <앱명>
scripts/run-k8s/main-makemigrations.sh
# 개발 환경에 마이그레이션 적용
scripts/run-k8s/main.sh python manage.py migrate
```
