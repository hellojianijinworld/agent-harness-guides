# Factory 작성 규칙

Factory를 정의·수정할 때 읽는다. 기존 Factory 호출만 사용하는 테스트에는 [테스트 공통](testing.md)의 데이터 준비 규칙이면 충분하다.
공유 fixture도 정의·수정하면 [Fixture](fixture.md)를 추가로 읽는다.

## 위치와 기본값

- `apps/<앱>/factories.py`에 `factory.django.DjangoModelFactory`를 정의하고 `Meta.model`을 지정한다.
- 기본값은 정상 도메인 데이터를 만들어야 한다. DB에 저장된다는 것만으로 업무 규칙을 만족한다고 간주하지 않는다.
- FK에는 관련 모델 인스턴스 또는 `SubFactory`를 사용한다. FK 이름에 연도 같은 표시값을 넘기지 않는다.
- 다른 앱의 `SubFactory`는 문자열 경로, 같은 앱은 클래스 참조를 사용한다. 순환 참조가 생기면 의존 관계부터 확인한다.

```python
import factory
import factory.django

from .models import CreativeActivity


class CreativeActivityFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = CreativeActivity

    activity_type = "autonomous"
    year = factory.SubFactory("apps.schools.factories.AcademicYearFactory")
    teacher = factory.SubFactory("apps.users.factories.TeacherFactory")
    name = ""
    grade = 1
    class_num = 1

    class Params:
        club = factory.Trait(
            activity_type="club",
            name=factory.Sequence(lambda n: f"동아리 {n}"),
            grade=None,
            class_num=None,
        )
```

실제 모델 필드와 관계명은 [기존 factories.py](../apps/creative_activities/factories.py)를 확인한다. 멤버의 관계명은 `activity`이며, Record에는 `member`·`content`가 있다.

## 관계와 변형

| 필요 | 선택 |
| --- | --- |
| 유일성이 필요한 값 | `factory.Sequence`; 복합·조건부 제약의 실제 조합도 확인 |
| 다른 필드에 의존하는 값 | `factory.LazyAttribute` |
| 반복되는 의미 있는 상태 변형 | `factory.Trait`; 한 필드 변경은 호출 시 override로 충분 |
| 저장 후 관계 생성 | `post_generation`; `create`·`extracted`를 확인하고 필요한 관계만 생성 |

- 같은 소유자·학년도여야 하는 객체는 이미 만든 인스턴스를 공유한다. 독립적인 SubFactory 기본값에 정합성을 맡기지 않는다.
- `Factory()`는 저장, `create_batch(n)`은 여러 개 저장, `build()`는 비저장 객체 구성이다. `build()`에서도 custom hook이 DB를 쓰는지 확인한다.
- 공유 데이터 구성이 반복될 때만 fixture로 추출한다. 준비 데이터의 `Model.objects.create()`는 기존 Factory로 대체하되, ORM 동작 자체를 검증하는 실행 단계에는 해당 ORM을 직접 호출한다.
