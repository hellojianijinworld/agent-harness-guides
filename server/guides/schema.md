# 스키마 작성 규칙

요청·응답 타입, 직렬화, 입력 형태 변환을 변경할 때 읽는다.
DB 상태·권한·조건부 업무 검증도 변경하면 [서비스와 API](service-layer.md)를 추가로 읽는다.

## 역할과 필드

- 입력은 Django Ninja `Schema`, 모델 기반 출력은 `ModelSchema`를 사용한다. 스키마는 타입·필드 제약·입출력 표현을 담당한다.
- 선택 필드는 `Field(default, description="...")`로 기본값과 의미를 명시한다. `T | None`이라는 타입과 필드 생략 허용 여부는 별개다.
- 타입·enum·`Field`로 표현할 수 있는 제약은 스키마에서 검증한다. 모델에 같은 제약이 있으면 `_meta.get_field()`로 `max_length`, `max_digits` 등을 참조한다. API에 별도 제약이 필요하면 이유를 명시한다.

```python
from ninja import Field, Schema

from .models import ActivityType, CreativeActivity


class CreativeActivityCreateIn(Schema):
    activity_type: ActivityType
    year: int = Field(..., description="학년도")
    name: str = Field(
        "", max_length=CreativeActivity._meta.get_field("name").max_length,
        description="활동명 (동아리 필수)",
    )
    grade: int | None = Field(None, ge=1, le=6, description="학년")
    class_num: int | None = Field(None, ge=1, description="반")
```

동아리의 이름 필수 여부처럼 조건부 업무 규칙은 서비스에서 검증한다.

## 입력 형태 변환

멀티파트의 JSON 문자열을 목록으로 파싱하는 등 입력 표현을 타입에 맞추는 변환은 스키마에 둔다. `model_validator`의 `before`·`after`는 필요한 입력 형태에 맞춰 고른다. 실행 시점만으로 업무 검증의 위치를 결정하지 않는다.

- `before`의 입력은 검증된 모델 인스턴스가 아니다. 해당 호출 경로의 dict·멀티파트 입력 구조를 확인하고 접근한다.
- JSON 디코딩 실패 등 입력 형태 오류는 기존 API 계약에 맞춰 `ninja.errors.HttpError(400, ...)`로 처리한다. 파싱 후 항목 타입·필드 제약 위반은 스키마 검증에 맡긴다.
- 파싱 예제는 [CreativeActivityBatchSplitPdfUploadIn](../main/apps/creative_activities/schema.py)의 구현을 참고한다. 해당 멀티파트 전용 접근 방식을 일반 JSON 입력에 복사하지 않는다.

## 출력 표현

`resolve_*`는 전달된 객체의 속성을 출력 형태로 변환한다. 관련 객체 접근에 필요한 eager loading은 호출 측에서 준비한다.

```python
@staticmethod
def resolve_has_content(obj):
    record = getattr(obj, "record", None)
    return bool(record and record.content)
```

스키마 안에서 `Model.objects.filter()`, `select_related()`, `prefetch_related()`, `order_by()`나 prefetch 유무 분기를 추가하지 않는다. 조회 구성은 뷰·서비스·QuerySet에 둔다.
데이터 변경, `transaction.atomic`, 업무용 `create()`·`update()`·`process()`도 스키마 책임이 아니다.

## 오류 계약

- **422**: 타입·enum·Field 제약 등 스키마 검증 오류.
- **400**: 입력 형태 변환 실패 또는 서비스의 업무 검증 오류. `HttpError`는 스키마에서 입력 형태 오류에만 사용한다.
- 인증·권한·존재 여부는 API·서비스 경계에서 기존 401·403·404 계약을 유지한다.

기존 구현과 새 규칙이 다르면 현재 변경에 필요한 부분만 다룬다. 스키마 정리 과정에서 상태 코드나 출력 계약을 임의로 바꾸지 않는다.
