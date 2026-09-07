# Fixture 탐색과 작성

fixture를 찾거나 정의·재정의할 때 읽는다. 알고 있는 fixture를 테스트 인자로 사용하는 것만으로 이 문서를 다시 읽을 필요는 없다.
Factory 정의도 바꾸면 [Factory 작성](factory.md), DB fixture를 선택하면 [DB fixture](testing-db.md)를 추가로 읽는다.

## 탐색과 위치

1. `server/main/_test_fixture/`에서 전역 fixture를 검색한다. 아래 표는 탐색용 요약이며 실제 정의가 기준이다.
2. 대상 테스트에서 가까운 `conftest.py`·파일·클래스의 같은 이름 fixture를 확인한다. 로컬 재정의가 전역 fixture보다 우선한다.
3. 없으면 사용하는 가장 좁은 범위에 정의한다. 한 클래스는 클래스 내부, 한 파일은 모듈 수준, 앱의 여러 테스트 파일에서 공유하면 `apps/<앱>/tests/conftest.py`에 둔다. 전역 재사용이 필요할 때만 `_test_fixture/`에 둔다.

클래스 fixture는 첫 인자로 `self`를 받는다. 테스트 함수 내부에 fixture를 중첩 정의하지 않는다.
`server/main/conftest.py`는 아래 6개 파일을 import해 전역에 노출한다. 파일을 전부 읽기보다 fixture 이름·도메인으로 검색한 뒤 필요한 정의와 의존 fixture를 읽는다.

| 파일 (`server/main/_test_fixture/`) | 찾을 내용 |
| --- | --- |
| `client.py` | `client`, `teacher_client`, `other_teacher_client`, `other_school_teacher_client`, `unauthorized_teacher_client`, `admin_teacher_client`, `other_admin_teacher_client`, `multi_subject_category_teacher_client`, `student_client`, `admin_client`, `staff_client`, `client_factory` |
| `setup_data.py` | 교사·학생·학교, `academic_year`·`last_academic_year`, `academic_semester`·`last_academic_semester`, `base_students` |
| `mocked.py` | `mock_slack`, `mock_lock`, reCAPTCHA mock |
| `utils.py` | `b64uuid`, `capture_queries_context` |
| `settings.py` | autouse DB 접근·기본 데이터, `enable_transactional_db_access`, 임시 저장소 |
| `rubric.py` | 평가 기준 데이터 |

## 프로젝트 고유 동작

- `teacher_client.user.teacher`가 로그인 교사다. 기본 fixture로 부족하면 `client_factory(user=teacher.user)`로 클라이언트를 만든다.
- `other_teacher`는 같은 학교, `other_school_teacher`는 다른 학교의 교사다. 권한 테스트에서 구분한다.
- `base_students`는 미리 만든 객체를 key로 조회한다. 임의 학번을 생성하지 않는다. 학생 key는 `student_GCCNN` 형식이며 현재 G=1~3, CC=01, NN=01~03을 제공한다. 인자 하나면 객체 하나, 여러 개면 tuple을 반환한다.

```python
students = base_students("student_10101", "student_10102")
```

- `mock_slack`은 알림 전송을 자동 mock한다. `mock_lock`은 cache backend에 `lock`이 없을 때만 대체하므로, 동시성 테스트에서는 실제 사용하는 backend와 lock 구현을 확인한다.
- `capture_queries_context(n)`은 전체 SQL 수가 아닌, 값을 정규화한 동일 SQL 패턴의 최대 반복 횟수를 검사한다 (기본 1). 전체 쿼리 수의 계약이 필요하면 `django_assert_num_queries` 등을 사용한다. 관련 예제는 N+1 테스트를 작성할 때 [엔드포인트 테스트](testing-endpoint.md)를 참고한다.

## 작성과 정리

```python
import pytest

from apps.creative_activities.factories import CreativeActivityFactory


@pytest.fixture
def creative_activity(teacher, academic_year):
    return CreativeActivityFactory(teacher=teacher, year=academic_year)
```

- DB 레코드는 테스트 DB 격리로 정리한다. 캐시·파일·외부 상태는 `yield` 뒤에서 정리한다.
- `autouse=True`는 해당 범위의 모든 테스트에 필요한 부수효과에만 사용한다. 앱 전용 mock을 전역으로 올리지 않는다.
- 동일한 상태 변형을 여러 테스트가 공유할 때 fixture `params`를 사용한다. 모든 소비 테스트의 실행 횟수가 늘어난다는 점을 확인한다.
- 입력값 몇 개만 다른 경우는 `@pytest.mark.parametrize`로 충분하다. 준비 과정이 서로 다르고 같은 동작 계약을 검증할 때만 상황별 fixture와 `request.getfixturevalue(request.param)`을 사용한다.
- 모듈 전체에 같은 부수효과 fixture가 필요하면 `pytestmark = pytest.mark.usefixtures("fixture_name")`을 사용한다.
