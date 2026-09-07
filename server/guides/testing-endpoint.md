# 엔드포인트 테스트 패턴

HTTP 엔드포인트 테스트를 작성·수정할 때 [테스트 공통](testing.md)과 함께 읽는다.
fixture 탐색·작성은 [Fixture](fixture.md), 커밋·동시성 검증은 [DB fixture](testing-db.md)를 해당 작업에서만 추가한다.

## 배치와 URL

- 신규 테스트는 `tests/endpoints/test_<endpoint>.py`에 엔드포인트별로 배치하고 `__init__.py`를 만든다. 같은 엔드포인트의 시나리오는 클래스에 묶고 클래스명에 대상을 드러낸다.
- 기존 테스트 보강은 현재 파일에서 한다. 요청과 무관한 파일 이동은 하지 않는다. `tests/conftest.py`와 `tests/models/`는 별도 역할을 유지한다.
- 요청 URL은 `get_url()`에서 `reverse()`로 생성한다. 경로 인자는 `args=[key, ...]`로 전달하고, 인자가 없으면 생략한다.
- 각 엔드포인트 파일에 경로 계약을 확인하는 `test_url`을 둔다. 클래스 밖의 함수로 작성해 클래스 autouse fixture를 피한다. 경로 확인용 key는 DB 객체 대신 `b64uuid` fixture를 쓴다.

```python
from django.urls import reverse


def get_url(key):
    return reverse("api-1:get_creative_activity", args=[key])


def test_url(b64uuid):
    assert get_url(b64uuid) == f"/api/v1/creative-activities/{b64uuid}"
```

경로 인자가 여러 개면 포맷 확인에 같은 `b64uuid`를 재사용할 수 있다. `self.url`이 필요하면 클래스 fixture에서 `get_url(object.key)`로 구성한다.

## 권한과 입력 검증

- 허용하지 않는 역할은 `request.getfixturevalue`로 기존 client fixture를 parameterize한다. 교사 전용 API의 비로그인·학생 등은 기존 401 계약을 검증한다.
- 같은 학교의 다른 교사, 다른 학교 교사, 존재하지 않는 key를 구분한다. 소유 리소스를 숨기는 API는 404 계약을 검증한다. 거부된 쓰기는 데이터도 바뀌지 않았는지 확인한다.
- 정상 payload에서 검사 대상만 변경한다. 다른 필수값 누락으로 우연히 실패하지 않도록 한다.
- 422는 타입·enum·Field 제약이다. 상태와 함께 실패한 필드·원인을 확인한다. 조건부 필수값·중복 등 업무 검증은 400이다.

```python
import pytest


@pytest.mark.parametrize("field,value", [("activity_type", "unknown"), ("grade", 0)])
def test_필드_제약(self, teacher_client, academic_year, field, value):
    payload = {
        "activity_type": "autonomous", "year": academic_year.year,
        "grade": 1, "class_num": 1,
    }
    payload[field] = value
    response = teacher_client.post(self.url, payload, content_type="application/json")
    assert response.status_code == 422
    assert any(error["loc"][-1] == field for error in response.json()["detail"])
```

위 메서드는 엔드포인트 테스트 클래스 안에 둔다. 동아리 이름 누락을 테스트하려면 `year`를 포함한 동아리 payload에서 `name`만 생략하고 **400**과 업무 에러 응답을 검증한다.

## 쿼리 성능 (N+1 검증)

- 목록에 여러 부모·관계 객체를 준비하고 실제 응답 내용도 검증한다. 빈 목록이나 객체 하나만으로 N+1을 검증하지 않는다.
- `capture_queries_context()`는 정규화한 동일 SQL 패턴이 반복되는지 검사한다. 기본값 1은 전체 SQL 수가 아니다. 필요한 반복만 인자로 허용한다.
- 전체 쿼리 수가 계약이면 `django_assert_num_queries` 같은 별도 도구를 사용한다. 검사 구간에 대상 조회·직렬화를 포함한다.
