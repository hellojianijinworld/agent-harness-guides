# 모델 작성 규칙

모델·관계·제약·조회 프로퍼티·QuerySet을 변경할 때 읽는다.
비즈니스 검증·데이터 변경 로직까지 다루면 [서비스와 API](service-layer.md), API 타입도 바꾸면 [스키마](schema.md)를 추가로 읽는다.

## 책임과 베이스

- 모델은 데이터 구조와 상태 조회·표현을 담당한다. 비즈니스 검증, `HttpError`, 업무상 `create()`·`update()`·트랜잭션 조율은 서비스에 둔다.
- 도메인 모델은 `config.models.TimestampedModel`을 상속한다. `created_at`, `updated_at`, 외부 노출용 22자 `key`를 제공한다. `key`는 내부 PK와 별개이며 API에서는 `B64uuid` 타입으로 표현한다.
- AI 결과 모델은 `AIResultBase` 또는 `AIResultBaseForTextType`을 상속한다.
- 스키마 변경은 마이그레이션으로 반영한다. 생성 명령은 서버 진입점의 관리 명령을 따른다.

## 필드·관계·Meta

- 필드의 `verbose_name`은 `_()`로 감싼 한국어 이름을 사용한다. 첫 인수 또는 키워드로 지정한다.
- FK/OneToOne에는 `related_name`을 지정한다. 다른 앱 모델은 문자열 경로로 참조한다.
- `CASCADE`: 부모와 함께 삭제. `PROTECT`: 참조 중인 부모 삭제 차단. `SET_NULL`: 부모 삭제 후 자식 보존 (`null=True` 필요).
- `TextChoices`는 여러 모델이 공유하면 모듈 레벨에 둔다. 한 모델 전용이면 내부 선언도 허용한다.
- `Meta`에 번역된 `verbose_name`과 `verbose_name_plural`을 지정한다. 복수명은 `" 목록"` 패턴을 사용한다.
- 유일성은 `UniqueConstraint`로 DB에도 선언한다. 자주 조회하는 필드 조합에 index를 검토하고, 기본 순서가 명확할 때만 `ordering`을 지정한다.

```python
from django.db import models
from django.utils.translation import gettext_lazy as _

from config.models import TimestampedModel


class CreativeActivity(TimestampedModel):
    class Meta:
        verbose_name = _("창의적 체험활동")
        verbose_name_plural = _("창의적 체험활동 목록")

    year = models.ForeignKey(
        "schools.AcademicYear",
        verbose_name=_("학년도"),
        on_delete=models.PROTECT,
        related_name="creative_activities",
    )
    # 나머지 필드·제약은 실제 모델 참고
```

실제 필드·제약의 예: [CreativeActivity](../main/apps/creative_activities/models.py).

## 조회와 프로퍼티

- 반복되는 조회는 QuerySet으로 분리하고 `as_manager()`로 연결한다. `get_by_natural_key()` 같은 Manager 전용 동작이 필요하면 Manager를 사용한다.
- `__str__`를 정의하되 자기 필드만 사용한다. `self.teacher.name`처럼 FK를 따라가면 목록에서 N+1을 만들 수 있다.
- 가벼운 계산은 `@property`, 반복되는 비싼 조회·계산은 `@cached_property`를 검토한다. 캐시 값은 같은 인스턴스의 데이터·관계가 바뀌어도 자동으로 최신이 되지 않는다. 변경 후 다시 읽어야 하면 무효화하거나 캐시하지 않는다.
- 관계를 읽는 프로퍼티는 호출 측의 `select_related`·`prefetch_related`와 함께 검토한다. 캐시는 N+1 해결책을 대신하지 않는다.
- 파일이 길다는 이유만으로 Mixin을 추가하지 않는다. 기존 QuerySet·서비스로 책임을 나누고, 기존 Mixin이나 실제 공통 기능이 있을 때 재사용한다.

## 타입 힌트와 순환 import

타입 힌트에만 필요한 클래스는 `TYPE_CHECKING`에서 import하고 문자열 어노테이션을 사용한다.

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from apps.ai_pipelines.models import CourseRecordAIPipeline


def get_pipeline() -> "CourseRecordAIPipeline | None":
    ...
```

`from __future__ import annotations`를 쓰면 따옴표를 생략할 수 있지만, 일반 import의 실행을 막지는 않는다. 타입 전용 import에는 여전히 `TYPE_CHECKING`이 필요하다.
`isinstance` 등 런타임에서 클래스가 필요하면 실제 import가 필요하다. 순환이 생기면 필요한 함수 안으로 import를 옮기거나 의존 관계를 정리한다.

위치: 모델은 `models.py`, 반복 조회는 `querysets.py`. 타입 힌트를 위해 모델 파일 사이의 런타임 의존성을 새로 만들지 않는다.
