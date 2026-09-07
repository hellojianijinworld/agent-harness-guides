# DB fixture 가이드 — `db` vs `transactional_db`

`on_commit`, atomic 예외 이후 조회, 실제 커밋·동시성을 검증할 때 [테스트 공통](testing.md)과 함께 읽는다. 일반 CRUD 테스트에는 추가로 읽을 필요가 없다.

## 원칙

기본값은 `db`다. 이 프로젝트에서는 `enable_db_access`가 autouse로 적용되므로 테스트마다 직접 선언하지 않는다.

`transactional_db`는 실제 커밋이나 여러 DB 연결 사이의 가시성을 검증할 때만 사용한다. 일반 CRUD나 예외 검증에 사용하면 불필요하게 테스트가 느려진다.

| 상황 | 선택 |
| ---- | ---- |
| 일반 CRUD·조회·API 테스트 | 기본 `db` |
| `on_commit` 콜백 등록·실행 검증 | `django_capture_on_commit_callbacks` |
| `task_always_eager=True` Celery 테스트 | 기본 `db` |
| 실제 worker·별도 thread/process가 생성한 데이터를 조회 | `enable_transactional_db_access` |
| 동시성·락·실제 commit/rollback 경계 검증 | `enable_transactional_db_access` |

## `on_commit` 검증

`transactional_db` 대신 `django_capture_on_commit_callbacks`를 사용한다. `execute=True`이면 실제 DB commit 없이 context 종료 시 캡처한 callback을 실행해 커밋 이후 side effect를 검증한다. 실제 commit 경계나 다른 연결에서의 가시성을 검증하는 방식은 아니다.

```python
def test_알림을_커밋_후_전송(
    teacher_client,
    django_capture_on_commit_callbacks,
    mock_send,
):
    with django_capture_on_commit_callbacks(execute=True) as callbacks:
        response = teacher_client.post(url, payload)

    assert response.status_code == 201
    assert len(callbacks) == 1
    mock_send.assert_called_once()
```

## 실제 트랜잭션이 필요한 테스트

프로젝트 fixture인 `enable_transactional_db_access`를 테스트 단위로 명시한다.

```python
def test_동시_요청_멱등성(enable_transactional_db_access):
    ...
```

파일이나 클래스 전체에 적용해야 할 명확한 이유가 없다면 테스트 하나에만 사용한다.

## atomic 예외

atomic block 내부에서 발생한 예외는 block 밖에서 잡는다. savepoint가 rollback된 뒤에는 기본 `db`에서도 후속 조회가 가능하다.

```python
with pytest.raises(IntegrityError):
    with transaction.atomic():
        service.run()

record.refresh_from_db()
```

깨진 atomic block 안에서 쿼리하면 `TransactionManagementError`가 발생하며, `transactional_db`로 바꿔도 해결되지 않는다. 예외 처리 경계를 먼저 수정한다.
