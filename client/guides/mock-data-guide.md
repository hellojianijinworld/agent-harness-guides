# 프론트엔드 Mock 데이터 가이드

API 응답 Mock이나 관계형 테스트 데이터를 구성·변경할 때 읽는다. 테스트 코드도 수정하면 [테스트 패턴](testing-patterns.md)을 함께 읽는다.

## 역할 구분

- 반환 데이터는 `src/test/mocks/schemas.ts`의 `PascalCaseMock`을 기반으로 만든다.
- 테스트 분기와 객체 관계에 영향을 주는 값은 명시적으로 고정한다.

`schemas.ts`, `response/`, `handlers/`, `fakers.ts`, `mockHandlers.ts`는 OpenAPI에서 생성되므로 직접 수정하지 않는다. generator 입력 설정인 `specials/`와 `server.ts`, `components.tsx`, `useGlobalSetting.ts` 같은 수동 테스트 파일은 이 규칙의 대상이 아니다.

## 기본 패턴

생성 Mock을 spread하고 테스트에 필요한 값만 덮어쓴다. 가능하면 `as Type` 대신 `satisfies Type`으로 결과 타입을 확인한다.

```typescript
import type { ProjectListOut } from '@/apis'
import { ProjectListOutMock } from '@/test/mocks/schemas'

const project = {
  ...ProjectListOutMock,
  key: 'project-1',
  has_homework: true,
  is_temporary: false,
  is_done_review: false,
} satisfies ProjectListOut
```

생성 Mock 자체는 API 타입을 명시적으로 참조하지 않을 수 있다. spread만으로 타입 안전성이 완성된다고 가정하지 않는다.

## 결정적인 값

생성 Mock에는 faker 값이 포함된다. 결과에 영향을 주는 key·참조, boolean·enum, 배열 길이·순서, 날짜·학년도·학년·반, 정렬·비교 값은 명시적으로 고정한다.

## 관계 정합성

- 참조 key를 따로 복제하기보다 이미 만든 객체를 재사용한다.
- 부모를 먼저 만들고 자식이 부모 값을 참조하게 구성한다.
- 의도적으로 불일치하는 데이터는 객체마다 한 가지 조건만 다르게 하고 이유를 주석으로 남긴다.

```typescript
const course = {
  ...CourseDetailOutMock,
  key: 'course-1',
  grade: 1,
  is_group_mode: false,
}

const students = [
  { ...StudentOutMock, key: 'student-1', grade: 1 },
  { ...StudentOutMock, key: 'student-2', grade: 2 }, // 학년 제외 조건
]

const member = {
  ...CourseMemberOutMock,
  key: 'member-1',
  student: students[0],
}
```

## 중첩 값 복사

`{ ...Mock }`은 얕은 복사다. 중첩 객체나 배열을 수정할 때는 해당 값도 새로 만든다. 공유된 생성 Mock을 직접 변경하면 다른 테스트 데이터까지 오염될 수 있다.

```typescript
const project = {
  ...ProjectDetailOutMock,
  elements: ProjectDetailOutMock.elements.map((element) => ({
    ...element,
    levels: element.levels.map((level) => ({ ...level })),
  })),
}
```

## 최소 데이터와 재사용

- 한 테스트에서 한 번 쓰는 데이터는 테스트 가까이에 둔다.
- 여러 테스트에서 같은 관계 구성이 반복될 때만 작은 builder나 fixture 함수로 추출한다.
- builder는 시나리오 이름을 드러내고 필요한 override만 받는다. 모든 필드를 선택적으로 받는 범용 factory를 만들지 않는다.
- API 응답 전체가 필요하지 않은 순수 함수 테스트라면 작은 직접 객체가 더 명확할 수 있다. 필요한 타입을 `satisfies`로 확인한다.
