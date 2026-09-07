# 프론트엔드 테스트 패턴

컴포넌트·hook·유틸 테스트를 작성·수정할 때 읽는다.
API 응답 Mock·관계형 데이터를 구성·변경하면 [Mock 데이터](mock-data-guide.md)를 추가로 읽는다. 테스트만 수정할 때 React 구현 가이드는 필요하지 않다.

## 기본과 검증 대상

- Vitest의 `test`를 사용한다. `test`, `expect`, `describe`, `beforeEach`, `afterEach`, `vi`는 global이므로 import하지 않는다. 타입은 `import type`을 사용한다.
- Provider가 필요하면 `render`, `renderHook`, `screen`, `waitFor`, `userEvent`를 `@/test/test-utils`에서 가져온다. Provider가 필요 없는 순수 hook은 `@testing-library/react`의 `renderHook`을 사용할 수 있다.
- 실행은 `--run`을 사용한다. 동기 유틸 테스트는 불필요하게 `async`로 만들지 않는다.
- 이름과 assertion으로 현재 동작 계약을 설명한다. 과거 수정 과정이나 특정 구현의 부재만 증명하는 테스트는 남기지 않는다.
- 사용자에게 보이는 내용·가능한 동작·접근성·실제 side effect를 검증한다. role·label·text로 조회하고 적합한 의미가 없을 때 `data-testid`를 쓴다.
- 입력은 `userEvent`를 우선하며 파일·날짜·저수준 이벤트 등 표현하기 어려운 경우에 `fireEvent`를 쓴다.
- 컴포넌트 통합 테스트의 toast는 실제 UI로 검증한다. hook·utility에서 toast 호출 자체가 결과라면 method·payload를 spy할 수 있다.
- 제어 가능한 자식은 함께 렌더링한다. 무겁거나 범위 밖인 외부 의존성만 mock한다.

## 비동기 UI와 격리

- Query의 첫 결과는 `findBy*`·`waitFor`로 기다린다. loading·error·empty·success가 기능상 중요하면 각각 검증한다.
- 공유 `queryClient`는 테스트 사이에 `clear()`한다. 영속 cache도 사용하는 경우 `@/test/mockDatas`의 `resetQueryClient()`를 사용한다.
- 같은 렌더·사용자 흐름의 결과는 기존 테스트에 assertion을 보강한다. Given·사용자 흐름·기대 동작이 독립적이면 테스트를 나눈다.
- 전역 setup은 DOM·MSW handler·toast·Router mock 등을 정리한다. `vi.clearAllMocks()`는 호출 기록만 지우므로 테스트가 만든 응답 spy의 구현은 별도로 `mockRestore()`한다.
- 자신이 만든 spy만 복구한다. 전역 mock을 일괄 해제해 다른 테스트의 기본 설정을 없애지 않는다. 직접 바꾼 timer·전역 상태도 원복한다.

### 부재 검증

대상 요소의 표시 여부가 결정되는 상태까지 렌더링을 기다린 후 부재를 검증한다. 완료 조건은 API 결과 화면, 사용자 동작 결과, 로딩·전환 완료 등 해당 상태를 입증하는 것으로 정한다. 동기 주입도 타깃 화면이 렌더됐는지 확인한다.

```tsx
render(<Page />)
await screen.findByRole('heading', { name: '파일 제출' }) // 타깃 상태에서만 나타남
expect(screen.queryByRole('dialog', { name: '과제 시작 안내' })).not.toBeInTheDocument()
```

`waitFor(() => expect(queryBy...()).not...)`만으로는 첫 로딩 순간에 통과할 수 있다. 선택자를 `data-testid`로 바꿔도 완료를 보장하지 않는다.
“한 번도 렌더링되지 않음”이 계약이면 최종 DOM만 보지 않고 완료 조건까지 대상 렌더 호출도 감시한다.

## Router

현재 [vitest.setup.ts](../src/test/vitest.setup.ts)의 대체 범위를 고려한다.

| hook | 현재 테스트 동작 |
| --- | --- |
| `useParams`, `useLocation` | 명시적 mock 값이 없으면 실제 Router 사용 |
| `useNavigate` | 항상 `mockedFunctions.navigate` 반환; 실제 navigation을 실행하지 않음 |
| `useSearchParams` | 명시적 mock 값이 없으면 빈 `URLSearchParams`와 mock setter 반환 |

- route parameter·location은 `SingleMemoryRouter`의 `basePath`, `initialEntries`로 구성한다. 예: `basePath="/teachers/courses/:courseKey"`, `initialEntries={['/teachers/courses/course-1']}`.
- 이동 요청은 `mockedFunctions.navigate` 호출 인자로 검증한다. 검색 파라미터는 `mockedFunctions.useSearchParams.mockReturnValue([params, setter])`로 명시한다. 둘 다 `@/test/vitest.setup`에서 가져온다.
- Router와 무관한 좁은 단위 테스트는 `mockedFunctions.useParams.mockReturnValue({ courseKey: 'course-1' })`로 구성할 수 있다.
- 실제 이동 후 화면·URL 동작은 현재 wrapper만으로 검증되지 않는다. 실제 Router hook을 사용하는 테스트 설정이 필요하다. 개별 테스트에서 `react-router-dom` 전체를 다시 mock하지 않는다.

## API 응답

정상 응답 데이터는 생성 응답 함수를 spy한다. HTTP 오류·상태 코드·지연·요청별 응답은 `server.use`로 해당 handler만 재정의한다.

```typescript
import * as response from '@/test/mocks/response'
import type { MockInstance } from 'vitest'

let profileSpy: MockInstance<typeof response.getAppsUsersApiTeacherProfile200>
beforeEach(() => {
  profileSpy = vi.spyOn(response, 'getAppsUsersApiTeacherProfile200')
  // 각 테스트에서 profileSpy.mockReturnValue(profile)로 필요한 응답 설정
})
afterEach(() => profileSpy.mockRestore())
```

단일 테스트에서만 만든 spy는 `try/finally`로 복구해도 된다.
