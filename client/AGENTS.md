# Client 코딩 지침

파일명만으로 제한하지 않고 변경·검토하는 동작으로 가이드를 선택한다. 해당하는 문서는 각각 전체를 읽는다.

| 작업 | 읽을 가이드 |
| --- | --- |
| React 컴포넌트·hook·상태·API 연동 구현 | [React 프론트엔드](guides/react-frontend.md) |
| 서버 API 계약을 타입·Zod·Mock 생성물에 반영, 생성 설정 수정 | [API 스키마 생성](guides/api-schema-generation.md) |
| 테스트 작성·수정 (`*.test.ts`, `*.test.tsx` 등) | [테스트 패턴](guides/testing-patterns.md) |
| API 응답 Mock·관계형 테스트 데이터 구성·변경 | [Mock 데이터](guides/mock-data-guide.md) |

## 공통과 검증

- TypeScript와 기존 프로젝트 패턴을 따른다. 테스트만 수정하면 React 가이드를 추가로 읽을 필요는 없다.
- `openapi.json`, `src/apis/`, `src/orval/`, `src/test/mocks/{schemas.ts,response/,handlers/,fakers.ts,mockHandlers.ts}`는 생성물이므로 직접 수정하지 않는다. 생성 설정과 수동 Mock 파일의 구분은 API 스키마 생성 가이드를 따른다.
- 서버 구현도 수정하면 [서버 지침](../server/AGENTS.md)을 읽는다.
- 코드 변경 범위에 맞춰 아래 검증을 실행한다. 문서만 변경했다면 링크·예제·실제 설정과의 일치를 확인한다.

```bash
# client/에서
npm run check-types
npm run lint
npm test -- --run <테스트 파일>
```
