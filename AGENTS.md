# AGENTS.md

Behavioral guidelines to reduce common LLM coding mistakes. Merge with project-specific instructions as needed.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

## 작업별 지침 읽기

- `client/` 작업 전 [client/AGENTS.md](client/AGENTS.md), `server/` 작업 전 [server/AGENTS.md](server/AGENTS.md)를 읽는다. 양쪽 작업이면 모두 적용한다. 하위 경로의 추가 `AGENTS.md`도 확인한다.
- 각 진입점에서 작업 동작에 해당하는 가이드만 선택해 파일 전체를 읽는다. 여러 조건에 해당하면 합집합을 읽고, 링크는 명시된 조건에 따라 추가한다. 선택한 문서는 함께 읽되 출력이 잘리면 나머지를 확인한다.
- 구현 예제 링크는 해당 패턴을 확인할 때만 필요한 심볼과 주변 코드를 읽는다. 연결된 소스 파일 전체를 일괄 로드하지 않는다.
- 같은 작업에서 이미 읽은 문서는 변경되거나 내용 확인이 다시 필요할 때만 읽는다. 작업 범위가 바뀌면 선택을 갱신하고, 완료 전 실제 변경 경로·동작에 필요한 가이드를 빠뜨리지 않았는지 확인한다.
- 새 규칙은 기존 가이드에 통합한다. 기존 코드와 차이가 있어도 요청과 무관한 구조 변경은 하지 않으며, 현재 작업에 영향을 주는 충돌은 알린다.

## Development Workflow Git Permission

When the Development Workflow plugin is used for an implementation request, staging and committing the files in the agreed current review unit are authorized without separate confirmation. Do not include unrelated changes.

The plugin may create, move, or delete only the workstream-scoped lightweight tag `local/<workstream-id>/review-requested` to reflect a GitHub-confirmed contiguous review-request boundary. It must never push this tag.

This standing permission does not authorize push, pull request creation, any other tags, amend, rebase, or moving commits between review units. Those actions still require an explicit user request.

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:

- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:

- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:

- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:

- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:

```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

---

**These guidelines are working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, and clarifying questions come before implementation rather than after mistakes.
