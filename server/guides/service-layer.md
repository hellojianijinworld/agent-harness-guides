# 서비스와 API 작성 규칙

API·권한·비즈니스 검증·데이터 변경·트랜잭션을 다룰 때 파일 위치와 무관하게 읽는다.
입출력 타입도 바꾸면 [스키마](schema.md), 트랜잭션 동작을 테스트하면 [DB fixture](testing-db.md)를 추가로 읽는다.

## 책임 경계

| 위치 | 역할 |
| --- | --- |
| 뷰 (`api*.py`) | Django Ninja 라우팅·인증·입출력 지정, 권한 범위의 객체 조회와 조회 구성, 서비스 호출 |
| 서비스 | DB 상태·조건부 업무 검증, 데이터 생성·수정·삭제, 트랜잭션 조율 |
| QuerySet | 재사용하는 조회·관계 로딩·정렬 |
| 스키마 | 타입·필드 제약·입출력 표현 |

- API 경로·이름은 기존 RESTful 규칙을 따른다. 새 업무 로직은 서비스에 둔다. 기존 `services.py` 또는 `services/` 배치를 따른다. 필요할 때만 도메인별 파일로 분리한다.
- 여러 메서드가 교사 등의 컨텍스트를 공유하면 인스턴스 클래스를 사용한다. 공유 컨텍스트가 없는 단일 동작은 함수로 작성할 수 있다.
- 뷰의 단순 조회를 서비스 호출로 감싸기 위한 위임 계층은 만들지 않는다.

## 권한과 오류

- 인증된 역할뿐 아니라 객체의 교사·학교 소유 범위도 확인한다. 기존 `_get_activity`처럼 소유 관계에서 조회하고, 자식 key 조회도 부모·소유자 조건을 함께 제한한다.
- 현재 `teacher_session_auth`의 `request.auth`는 User다. 교사 컨텍스트에는 `request.user.teacher`를 사용한다.
- `HttpError`는 `ninja.errors`에서 가져온다. 조건부 필수값·중복 등 업무 검증 오류는 **400**, 스키마 검증 오류는 **422**다. 서비스에서 422를 직접 발생시키지 않는다.
- 인증·권한·존재 여부의 기존 401·403·404 계약은 유지한다. 모든 오류를 400으로 바꾸지 않는다.

```python
@router.post("", response={201: CreativeActivityDetailOut})
def create_creative_activity(request, payload: CreativeActivityCreateIn):
    service = CreativeActivityService(teacher=request.user.teacher)
    return 201, service.create(payload)
```

실제 권한 조회와 연동은 [creative_activities/api.py](../apps/creative_activities/api.py), 업무 검증은 [CreativeActivityService](../apps/creative_activities/services/creative_activity/__init__.py)를 참고한다.

## 트랜잭션과 외부 부수효과

- 함께 성공·실패해야 하는 DB 변경은 서비스 진입점의 `transaction.atomic()`으로 묶는다.
- 중복 사전 조회만으로 동시 요청의 유일성을 보장하지 않는다. DB 제약을 함께 사용하고, `IntegrityError`를 변환할 때는 해당 atomic block 밖에서 잡는다.
- 커밋된 데이터를 읽어야 하는 작업 큐 발행·알림은 `transaction.on_commit`을 사용한다. 등록·실행 검증과 실제 커밋 경계 검증을 구분한다.
- 파일 저장·외부 API 호출은 DB rollback으로 취소되지 않는다. 실패 시 정리 또는 재시도 방식은 해당 작업의 기존 패턴을 따른다. 일반 조회에 불필요한 트랜잭션을 추가하지 않는다.
