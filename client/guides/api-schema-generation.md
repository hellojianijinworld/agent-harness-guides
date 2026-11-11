# API 스키마 생성 가이드

서버 API 계약을 클라이언트 타입·Mock·요청 검증 생성물에 반영하거나 생성 설정을 바꿀 때 읽는다.
생성 후 테스트 Mock 데이터를 직접 구성하거나 바꾸면 [Mock 데이터](mock-data-guide.md)를 추가로 읽는다.

## 실행

전체 생성은 `client/`에서 Node.js `22.22.2`로 `npm run api-gen`을 실행한다. 최초 실행 전에 해당 Node 버전을 설치한다.

```bash
nvm install 22.22.2
npm run api-gen
```

실행하기 전에 입력으로 사용할 OpenAPI가 현재 작업한 서버 코드에서 나온 것인지 확인한다. `SERVER_URL`을 사용할 때는 실행 중인 서버를 확인하고, `OPENAPI_PATH`를 사용할 때는 그 URL 또는 파일을 확인한다.

```bash
# SERVER_URL을 사용할 때
OPENAPI_URL="${SERVER_URL:-http://localhost:8000}/api/v1/openapi.json"
curl --fail --silent "$OPENAPI_URL" | jq -e '.openapi and .info and .paths' >/dev/null

# OPENAPI_PATH가 URL일 때
curl --fail --silent "$OPENAPI_PATH" | jq -e '.openapi and .info and .paths' >/dev/null

# OPENAPI_PATH가 로컬 파일일 때
jq -e '.openapi and .info and .paths' "$OPENAPI_PATH" >/dev/null
```

입력 방식에 맞는 명령 하나만 실행한다.

입력 경로를 바꾸거나 이미 받은 명세로만 다시 생성할 수 있다.

```bash
SERVER_URL=http://super-clipo.me npm run api-gen
OPENAPI_PATH=https://example.com/api/v1/openapi.json npm run api-gen
OPENAPI_PATH=./server_openapi.json npm run api-gen
npm run api-gen -- --local
```

`--local`은 서버에 요청하지 않고 현재 `openapi.json`을 사용한다. 기본 생성은 명세를 받아 `openapi.json`을 갱신한 뒤 모든 생성기를 순서대로 실행한다.

## 생성 순서와 보정

`api-gen.ts`는 다음 순서로 실행한다.

1. OpenAPI를 받아 JSON 구조(`openapi`, `info`, `paths`)를 확인하고 `openapi.json`에 저장한다.
2. `openapi-typescript-codegen`으로 `src/apis/`를 생성한다.
3. `openapi-ts-mock-generator`로 `src/test/mocks/`를 생성한다.
4. `scripts/fix-mock-response-imports.mjs`가 실제 `get...` export를 기준으로 Mock response index와 handler import를 보정한다. 생성기가 스키마 주석을 함수로 잘못 읽어 `getOptionsOut` 같은 import를 남기는 문제를 정리하는 단계다.
5. Orval로 `src/orval/`을 생성한다.
6. 생성된 파일에 Prettier를 적용한다.

한 단계라도 실패하면 이후 생성은 실행되지 않는다. 생성 후에는 변경된 API·Zod·Mock 범위가 요청한 계약과 일치하는지 확인한다.

## 생성물과 수정 위치

| 생성물 — 직접 수정 금지 | 설정·원본 수정 위치 |
| --- | --- |
| `openapi.json`, `src/apis/` | 서버 스키마, `scripts/api-gen.ts`, `src/request-template.ts` |
| `src/orval/` | `orval.config.ts` |
| `src/test/mocks/{schemas.ts,response/,handlers/,fakers.ts,mockHandlers.ts}` | `src/test/mocks/specials/`, 생성 스크립트 |

API 계약이나 생성 설정을 바꾼 뒤 전체 생성 명령을 다시 실행한다. 생성물에서 오류를 발견해도 생성 파일을 직접 고치지 않고 서버 스키마, 생성 설정, 보정 스크립트 중 원인을 수정한다.
`src/test/mocks/server.ts`, `components.tsx`, `useGlobalSetting.ts`처럼 수동으로 관리하는 테스트 파일은 생성물과 구분한다.

## 생성 후 확인

- 변경한 엔드포인트·스키마가 반영됐고, 관계없는 API가 대량으로 바뀌지 않았는지 확인한다.
- 생성 과정이 실패했으면 일부 파일만 바뀐 상태를 완료로 간주하지 않는다.
- `npm run check-types`와 영향받는 테스트를 실행한다.
