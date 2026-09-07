# 테스트 공통 가이드

서버 테스트를 작성·수정할 때 읽는다.
추가 필수 문서: HTTP 테스트는 [엔드포인트](testing-endpoint.md), 모델 제약·메서드 테스트는 [모델 테스트](testing-models.md), 트랜잭션 경계·atomic 예외 이후 조회는 [DB fixture](testing-db.md). 해당 조건만 추가한다.

## 준비와 실행

- `enable_db_access`가 autouse이므로 일반 테스트에 `@pytest.mark.django_db`나 `db` 인자를 추가하지 않는다.
- Given의 DB 데이터는 기존 fixture·Factory를 사용한다. 테스트 대상인 ORM·서비스 호출은 When에서 직접 실행한다.
- 기존 Factory는 `Factory(field=value)`, 여러 객체는 `create_batch(n, ...)`로 생성한다. 같은 소유자·학년도는 인스턴스를 공유하고 분기에 영향을 주는 값을 고정한다.
- Factory 정의를 변경할 때만 [Factory 작성](factory.md)을 읽는다. fixture를 찾거나 정의·재정의할 때는 [Fixture](fixture.md)를 읽고 기존 것을 먼저 확인한다.
- 외부 서비스(OpenAI, S3 등)는 mock·stub으로 대체한다. 시간 의존 테스트는 `freezegun`을 사용한다.
- 준비·실행·검증 단계가 있는 테스트는 `# Given` / `# When` / `# Then`으로 구분한다. 단순 URL 확인이나 예외 검증은 생략할 수 있다.

## 테스트 단위

- 이름과 assertion으로 현재 요구사항의 동작 계약을 설명한다. 과거 수정 과정이나 특정 구현의 부재만 증명하는 테스트는 남기지 않는다.
- 같은 실행 경로의 필드 검증은 기존 테스트를 보강한다. 같은 요청·목록에서 포함·제외를 함께 볼 수 있으면 한 Given과 한 요청으로 검증한다.
- 제외 데이터는 객체마다 제외 이유를 하나씩 다르게 한다. 요청 조건·사용자 흐름·기대 동작이 독립적이거나 페이지네이션 때문에 함께 볼 수 없으면 별도 테스트로 둔다.

## Assertion

- 개수만 확인하지 않고 실제 내용·소유자·연관 객체를 확인한다. 생성 응답의 `key`로 DB를 조회하거나 기대 key 집합을 비교한다.
- DB·외부 호출 등 실제 side effect를 검증한다. stub에 넣은 값을 다시 읽는 자기참조 assertion은 쓰지 않는다.
- 결과 전체를 통제하면 집합을 정확히 비교한다. 다른 정상 데이터가 공존하면 포함 집합은 부분집합, 제외 집합은 교집합이 없는지 확인한다.

```python
result_keys = {item["key"] for item in response.json()}
assert included_keys <= result_keys
assert excluded_keys.isdisjoint(result_keys)
```

업무 에러 응답은 `.get()`으로 일부만 읽지 않고 dict 전체를 비교한다.

```python
assert response.status_code == 400
assert response.json() == {"detail": "이미 등록된 학생입니다."}
```

## parametrize

| 상황 | 선택 |
| --- | --- |
| 입력값만 다르고 같은 동작·기대 결과 | `@pytest.mark.parametrize` |
| 준비 과정이 다르지만 같은 동작 계약 | 상황별 fixture; 필요하면 `request.getfixturevalue(request.param)`으로 순회 |
| 같은 요청에서 함께 관찰하는 포함·제외 | 한 테스트 |
| Given이나 기대 동작이 독립적 | 별도 테스트 |

파라미터에 따라 테스트 본문을 `if/elif`로 여러 시나리오로 나누지 않는다. 단순 값 차이를 위해 fixture를 여러 개 만들지도 않는다.

여러 상태를 위치 기반 tuple 하나에 압축하지 않는다. `field_states`처럼 `(True, False, False, ...)`만 보고 각 값이 어느 필드인지 기억해야 하는 패턴은 실패 원인과 리뷰 의도를 숨긴다.

- 기대 필드가 여러 개면 case별 기대값을 이름 있는 dict로 표현한다.
- 또는 `@pytest.mark.parametrize` 인자를 필드별로 펼쳐 이름이 assertion과 실패 메시지에 드러나게 한다.
- 입력·기대값이 단순한 1~2개 스칼라라면 기존 tuple parametrize를 그대로 사용한다. dict가 더 장황해지면 강제하지 않는다.
