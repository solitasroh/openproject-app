# CLAUDE.md

Behavioral guidelines to reduce common LLM coding mistakes. Merge with project-specific instructions as needed.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

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

## 5. 커밋 규칙 (Conventional Commits)

이 저장소의 커밋은 아래 규칙을 따른다. 상태 파악 → 논리 단위 분할 → 승인
게이트 → stage/commit → 검증으로 이어지는 전체 워크플로는 `/commit`
스킬(`.claude/skills/commit/SKILL.md`)에 정의되어 있으며, 커밋 의도가 보이면
그 스킬을 사용한다.

### 메시지 형식

`<type>(<scope>): <subject>` — 제목과 본문 사이 한 줄 공백, 이어서 본문.

- **type** (영문 고정): `feat`(기능) · `fix`(버그) · `docs`(문서만) ·
  `refactor`(동작 불변 구조 변경) · `test` · `chore`(설정·잡무) ·
  `style`(포맷) · `perf` · `build`(빌드·의존성) · `ci`. 변경 성격에서 고른다.
- **scope** (영문, 선택): 바뀐 모듈/영역을 경로에서 추론한다(예:
  `application`, `bootloader`, `tools`, `host-access`). 여러 영역에 광범위하게
  걸치면 생략한다. 최근 히스토리(`git log --oneline -20`)의 scope 어휘를
  재사용한다.
- **subject** (한국어): 무엇을 했는지 간결하게. 마침표 없음, 72자 이내.
- **body** (한국어, 선택): **why** 중심 1~2문장. *what* 은 diff 에 보이므로
  반복하지 않는다. 구현 내용만 적고 설계/plan 문서 참조는 금지한다. 자명한
  작은 변경은 본문을 생략한다.
- **footer 없음**: `Co-Authored-By`, `Claude-Session` 등 어떤 footer 도 붙이지
  않는다. author 는 사용자 본인만.

### OpenProject 티켓

사용자가 OpenProject 티켓 번호를 주면 subject 를
`[OP#<번호>]<type>: <한국어 subject>` 로 쓴다(`]` 뒤 공백 없음, 이때 scope 는
생략, prefix 포함 72자). 분할 커밋이면 모든 커밋에 같은 `[OP#<번호>]` 를
붙인다.

예: `[OP#1000]feat: DI debounce 임계값 채널별 적용`

### 절차 규칙

- **승인 게이트**: `git commit` 전 항상 최종 메시지 + stage 할 정확한 파일
  목록을 제시하고 승인받는다(AskUserQuestion).
- **명시 stage 만**: `git add .` / `git add -A` 는 쓰지 않는다. 각 커밋 단위의
  명시 파일 경로만 `git add -- <path>` 로 stage 한다. staged 된 게 있으면 그
  범위를 존중하고 추가로 stage 하지 않는다.
- **논리 단위 분할**: 무관한 관심사(기능 + 무관한 리팩터 + 문서 등)가 섞이면
  관심사별 커밋으로 분할한다.
- **위험 파일 제외**: 비밀/환경 파일(`.env`, `.env.*`, `*.pem`, `*.key`), 빌드
  산출물(`build/`, `dist/`, `*.bin`, `*.hex`, `*.map` 등), 대용량 바이너리는
  기본 제외하고 경고로 표시한다.
- **훅 존중**: `--no-verify` / `--no-gpg-sign` 으로 pre-commit 훅이나 서명을
  우회하지 않는다.

### 예시

```
feat(application): DI debounce setup extension 저장 지원

setup 확장 항목으로 채널별 debounce 값을 영속화해, 재기동 후에도
사용자 설정이 유지되도록 한다.
```

```
fix(boot): handoff 시 FTM0/FTM1만 deinit
```
