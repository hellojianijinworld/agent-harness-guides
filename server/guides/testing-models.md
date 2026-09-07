# 모델 테스트 패턴

HTTP 요청 없이 모델 제약·메서드·유효성 검증을 테스트할 때 [테스트 공통](testing.md)과 함께 읽는다.
IntegrityError 이후 DB를 조회하거나 실제 커밋·동시성을 검증하면 [DB fixture](testing-db.md)를 추가로 읽는다.

## 범위와 배치

- 단순 제약은 함수, 유사한 제약은 클래스로 묶는다.
- 신규 테스트는 `tests/models/test_<모델명_snake_case>.py`에 모델별로 배치하고 `__init__.py`를 만든다. 기존 테스트 보강은 현재 파일에서 하며 무관한 이동을 하지 않는다.
- 관계 테스트는 검증하려는 동작의 주체 모델 파일에 둔다.
- 필드 존재 여부나 `_meta.fields` 전체 집합을 선언대로 반복 검증하지 않는다. `_meta.get_field()`는 기본값·타입·help_text 자체가 명시적인 계약일 때만 사용한다.

## DB 제약과 모델 검증

복합 유일성은 같은 FK 인스턴스를 공유해 제약의 모든 필드를 맞춘다. Factory가 새 학년도를 만들면 중복 조건이 달라질 수 있다.

```python
import pytest

from django.db import IntegrityError, transaction

from apps.creative_activities.factories import CreativeActivityFactory


def test_동일_활동_중복_불가(teacher, academic_year):
    fields = dict(teacher=teacher, year=academic_year, grade=1, class_num=1)
    CreativeActivityFactory(**fields)
    with pytest.raises(IntegrityError):
        with transaction.atomic():
            CreativeActivityFactory(**fields)
```

다른 교사 등 제약 밖의 조합이 허용되는 정상 사례도 검증한다.
choices 등 모델 검증은 `full_clean()`으로 확인한다. 저장만으로 모델 검증이 모두 실행된다고 가정하지 않는다.

```python
from django.core.exceptions import ValidationError


def test_활동_유형_choices_유효성():
    with pytest.raises(ValidationError):
        CreativeActivityFactory(activity_type="invalid_type").full_clean()
```
