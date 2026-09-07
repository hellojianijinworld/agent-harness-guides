# React 프론트엔드 가이드

React 컴포넌트·hook·상태·API 연동을 구현·변경할 때 읽는다.
테스트도 수정하면 [테스트 패턴](testing-patterns.md), 생성물이 바뀌면 [API 스키마 생성](api-schema-generation.md)을 추가로 읽는다.

## 책임과 위치

- `src/pages/`는 route parameter·query·redirect·loading을 조율하고 화면은 기능 컴포넌트에 위임한다. 페이지 전용 UI는 기존 `src/components/<feature>-page/` 패턴을 따른다.
- 한 기능의 컴포넌트·hook·helper는 함께 둔다. 둘 이상 기능에서 실제 재사용할 때만 `components/common/`·`hooks/`로 올린다. Chakra wrapper는 `components/ui/`를 먼저 확인한다.
- 예상 재사용이나 줄 수를 기준으로 나누지 않는다. 독립 UI·별도 상태 흐름·반복 상호작용을 기준으로 분리한다.
- Props는 필요한 값·callback만 받는다. 도메인 객체가 입력이면 객체 전달도 허용하되 무관한 필드·상태까지 결합하지 않는다.
- 한 버튼으로만 여는 모달·드로어·팝오버는 버튼·열림 상태를 함께 관리한다. 여러 진입점이나 상위 흐름이 제어하면 공통 부모로 올린다. 예: [`WritingTraceHelpModalButton`](../src/components/writing-trace-page/WritingTraceHelpModalButton.tsx)처럼 사용자 진입점을 드러내는 이름을 사용한다.

## 상태와 데이터

- 서버 상태는 TanStack Query, 폼은 React Hook Form, 화면 내부 상태는 로컬 state를 우선한다. Recoil은 여러 화면·멀리 떨어진 컴포넌트의 공유 상태에 사용한다.
- API 서비스·요청·응답 타입은 `@/apis`, 요청 검증 Zod schema는 `@/orval`을 사용한다.
- 서버 리소스의 조회·invalidate·cache 접근은 `@/react-query-key`의 도메인 factory와 기존 `_ctx` 계층을 따른다. 다운로드 상태·화면 polling·fake query 등 서버 리소스 cache가 아닌 경우는 목적이 드러나는 로컬 key를 허용한다.
- query 결과를 별도 state에 복사하지 않는다. 파생 값은 렌더 중 계산하고, 비싼 계산만 `useMemo`를 검토한다. mutation 성공 후 관련 key만 invalidate하거나 갱신한다.
- API body 폼은 생성 Zod schema에 화면 고유 규칙만 `extend`·`refine`한다. 공유 폼은 상위 `FormProvider`와 하위 `useFormContext`로 연결한다.
- 여러 화면에서 재사용하는 조회·파생 로직은 `hooks/query/`, 기능 전용 polling·상태 전환·invalidate는 해당 기능의 `hooks/`에 둔다.
- 여러 조건이 결정하는 UI 상태는 순수 파생 함수로 분리해 검증할 수 있게 한다.

## UI와 접근성

- 비동기 화면의 loading·error·empty·success를 구분한다. loading에는 `role="status"` 등 의미를 제공한다.
- 권한·데이터 조건이 맞지 않으면 redirect 또는 명시적 empty/error UI를 제공한다. 조건이 불분명한 빈 화면을 반환하지 않는다.
- 중복 제출을 막고 mutation pending 상태를 버튼·피드백에 반영한다.
- Chakra·semantic token·`textStyle`·recipe를 우선하고 responsive 값은 기존 breakpoint 객체 패턴을 따른다. 임의 색상·간격·인라인 style은 기존 방식으로 표현할 수 없을 때 사용한다.
- 의미 있는 HTML·Chakra primitive를 사용하며 클릭을 일반 `div`에 넣지 않는다. 입력에는 label·오류 메시지를 연결하고 icon button에는 `aria-label`을 제공한다.
- 선택·확장·로딩은 `aria-pressed`, `aria-expanded`, `aria-selected`, `role="status"` 등 적합한 속성으로 노출한다.

## 성능

- 불필요한 요청·중복 state·과도한 렌더링·큰 목록 연산부터 줄인다.
- `React.memo`·`useMemo`·`useCallback`은 렌더 비용·참조 안정성 문제가 있을 때만 사용한다. `React.lazy`는 크고 드문 독립 화면처럼 번들 분리 효과가 분명할 때 사용한다.
- 성능 변경은 profiler·번들 분석·요청 수 등 확인 가능한 근거로 검증한다.
