# API 스키마 생성 가이드

서버 API 계약을 타입·Zod·Mock 생성물에 반영하거나 생성 설정을 변경할 때 읽는다.
생성 후 테스트의 Mock 데이터를 직접 구성·변경하면 [Mock 데이터](mock-data-guide.md)를 추가로 읽는다.

## 실행

서버 endpoint·요청·응답 schema가 바뀌면 클라이언트 타입이나 생성 파일을 수동 작성하지 않는다. 변경이 반영된 로컬 서버의 OpenAPI로 전체 생성물을 갱신한다.
`openapi-gen`은 nvm으로 Node 16과 22를 전환하므로 둘 다 설치되어 있어야 한다. 실행 중인 서버의 `/api/v1/openapi.json`이 변경한 코드의 schema인지 확인한다.

```bash
# client/에서
SERVER_URL=http://super-clipo.me npm run openapi-gen
```

## 생성물과 수정 위치

| 생성물 — 직접 수정 금지 | 설정·원본 수정 위치 |
| --- | --- |
| `openapi.json`, `src/apis/` | 서버 schema, `scripts/codegen.ts`, `src/request-template.ts` |
| `src/orval/`의 요청 검증 Zod schema | `orval.config.ts` |
| `src/test/mocks/{schemas.ts,response/,handlers/,fakers.ts,mockHandlers.ts}` | `package.json` 생성 명령, `src/test/mocks/specials/` |

`src/test/mocks/server.ts`, `components.tsx`, `useGlobalSetting.ts` 등 수동 테스트 파일은 생성 파일이 아니다.

## 생성 후 확인

- 변경 endpoint·schema가 반영됐고, 관계없는 API가 대량으로 바뀌지 않았는지 확인한다.
- 예상 밖 변경은 수동으로 정리하지 않고 `SERVER_URL`과 서버 OpenAPI부터 확인한다.
- 생성 과정이 실패했으면 일부 파일만 바뀐 상태를 완료로 간주하지 않는다. 원인을 해결한 뒤 전체 명령을 다시 실행한다.
- `npm run check-types`와 영향받는 테스트를 실행한다.
