---
name: commit
description: "현재 작업 변경을 Conventional Commits 규칙에 맞춰 논리 단위로 나누고, 한국어 subject·구현 중심 본문의 커밋 메시지를 작성해 승인 후 git commit 한다. 사용자가 '커밋해줘', '커밋해', '커밋', 'commit', 'commit this', '이거 커밋', '변경 커밋해' 처럼 커밋 의도를 보이면 반드시 이 스킬을 사용한다. staged 된 게 있으면 그것만, 없으면 작업트리 전체를 분석해 관심사별로 분할을 제안한다. 사용자가 OpenProject 티켓 번호를 주면 subject 를 `[OP#1000]feat: …` 형태로 쓴다. Co-Authored-By 등 footer 는 붙이지 않는다. '커밋 전에 리뷰/설명만', 'diff 만 보여줘', '아직 커밋하지 마'처럼 커밋을 실제로 만들지 말라는 요청에는 발동하지 않는다."
---

# commit

작업 변경을 Conventional Commits 규칙으로 커밋한다. 무엇을 커밋할지는 사용자의 작업 결과에 달려 있고, 이 스킬은 그것을 *어떻게* 논리 단위로 묶고, 어떤 메시지로, 어떤 승인 절차로 커밋하는지의 규율만 정의한다.

핵심 원칙: 커밋은 명시 요청이 있을 때만 만든다. 이 스킬이 호출된 것 자체가 그 요청이다. 그래도 실제 `git commit` 실행 전에는 항상 사용자 승인을 받는다 — 커밋은 되돌리기 번거로운 outward 동작이기 때문이다.

질의 방식: 사용자에게 확인·선택을 요청하는 모든 지점(승인 게이트, 분할·헝크 조율, 위험 파일 포함 여부 등)은 자유 서술 질문이 아니라 AskUserQuestion 툴(인터뷰 형식)로 묻는다 — 각 선택지에 추천안을 붙여서.

## 언제 / 언제 아닌가

- **호출:** 사용자가 커밋 의도를 보일 때(위 트리거) 또는 `/commit`.
- **호출 아님:** "커밋 전에 리뷰해줘", "diff 만 보여줘", "아직 커밋하지 마", "무엇을 커밋할지 설명만" — 커밋을 만들지 말라는 요청. 이때는 요청받은 것(리뷰·설명)만 하고 커밋하지 않는다.

## 0. 상태 파악 (커밋 전 항상)

1. `git rev-parse --is-inside-work-tree` — git repo 가 아니면 그 사실을 알리고 멈춘다.
2. `git status --porcelain=v1` + `git diff --stat` + `git diff --cached --stat` 로 staged / unstaged / untracked 를 구분해 파악한다.
3. 커밋할 변경이 하나도 없으면 "커밋할 변경이 없다"고 알리고 멈춘다. 없는 변경을 지어내지 않는다.
4. merge conflict, detached HEAD, rebase 진행 중 등 비정상 상태면 그 상태를 그대로 알리고 커밋을 강행하지 않는다.

## 1. 대상 범위 결정 (staged 우선)

- **이미 staged 된 파일이 있으면** 그것을 사용자가 의도한 커밋 범위로 존중한다. 추가로 stage 하지 않는다. staged 집합에 대해 §3 메시지를 작성하고 §4 로 간다. (unstaged/untracked 가 남아 있으면 "이번 커밋 밖에 남는 변경이 있다"고 한 줄 알린다.)
- **staged 가 하나도 없으면** 작업트리 전체(unstaged tracked + 관련 untracked)를 분석 대상으로 삼는다. `git add .` / `git add -A` 는 절대 쓰지 않는다 — §5 에서 명시 파일 경로만 stage 한다.

### untracked·위험 파일

전체 분석 모드에서 untracked 새 파일은 변경과 명백히 관련될 때만 후보에 넣는다. 다음은 기본 제외하고 목록에 경고로 표시한다(사용자가 명시하면 포함):

- 비밀/환경: `.env`, `.env.*`, `*.pem`, `*.key`, 자격증명 파일
- 빌드 산출물: `build/`, `out/`, `dist/`, `*.o`, `*.obj`, `*.elf`, `*.hex`, `*.bin`, `*.map`, `*.axf`
- 대용량 바이너리 / 아카이브

## 2. 논리 단위 그룹핑 & 분할

변경이 하나의 논리 단위면 단일 커밋으로 간다. 여러 무관한 관심사(예: 기능 추가 + 무관한 리팩터 + 문서)가 섞여 있으면 관심사별로 분할한다.

- 파일/헝크를 관심사로 묶어 커밋 N개의 **분할 계획**을 만든다. 각 단위는 자체 type·scope·subject·파일 목록을 가진다.
- 분할은 자동 실행하지 않는다. §4 에서 전체 계획을 한 번에 제시하고 승인받는다.
- 한 파일이 여러 관심사를 담고 있어 헝크 단위 분할이 필요하면 그 사실을 알리고, 헝크 stage(`git add -p`) 진행 여부를 AskUserQuestion 으로 조율한다.

## 3. 메시지 작성 규칙

형식: `<type>(<scope>): <subject>` — 제목과 본문 사이 한 줄 공백, 이어서 본문.

- **type** (영문 고정): `feat`(기능) · `fix`(버그) · `docs`(문서만) · `refactor`(동작 불변 구조 변경) · `test` · `chore`(설정·잡무) · `style`(포맷) · `perf` · `build`(빌드·의존성) · `ci`. 변경 성격에서 고른다.
- **scope** (영문, 선택): 바뀐 모듈/영역. 경로에서 추론한다 — 예 `products/application/**` → `application`, `products/bootloader/**` → `bootloader`, `tools/**` → `tools`, `Docs/host-access/**` → `host-access`. 여러 영역에 광범위하게 걸치면 생략한다. 최근 히스토리(`git log --oneline -20`)의 scope 어휘를 재사용한다.
- **subject** (한국어): 무엇을 했는지 간결하게. 마침표 없음, 72자 이내. 예: `DI debounce 설정 적용 경로 연결`, `program info 헤더 정렬로 bootable 통과`.
- **OP 티켓 (요청 시)**: 사용자가 OpenProject 티켓 번호를 주면(예 "OP#1000으로 커밋", "[OP#1000] 커밋해줘") subject 를 `[OP#<번호>]<type>: <한국어 subject>` 로 쓴다 — `]` 뒤 공백 없음, 이때 scope 는 생략한다(prefix 포함 72자). 번호를 주지 않으면 일반 형식(`<type>(<scope>): <subject>`)을 쓴다. 분할 커밋이면 모든 커밋에 같은 `[OP#<번호>]` 를 붙인다. 예: `[OP#1000]feat: DI debounce 임계값 채널별 적용`.
- **body** (한국어 허용, 선택): **why** 중심 1~2문장. *what* 은 diff 에서 보이므로 반복하지 않는다. 구현 내용만 적는다 — 설계/plan 문서 참조·환언 금지(`§`, "design", "Phase", "the plan says", 결정 이력). 자명한 작은 변경은 본문을 생략한다.
- **footer 없음**: `Co-Authored-By`, `Claude-Session` 등 어떤 footer 도 붙이지 않는다. author 는 사용자 본인만.

**예시**

```
feat(application): DI debounce setup extension 저장 지원

setup 확장 항목으로 채널별 debounce 값을 영속화해, 재기동 후에도
사용자 설정이 유지되도록 한다.
```

```
fix(boot): handoff 시 FTM0/FTM1만 deinit
```

## 4. 승인 게이트 (항상)

`git commit` 전에 사용자에게 제시하고 승인을 기다린다:

- 단일 커밋: 최종 메시지 + stage 할 정확한 파일 목록.
- 분할: 모든 커밋의 메시지 + 파일 목록을 담은 **전체 계획을 한 번에** 제시하고 전체를 승인받는다(커밋마다 개별 승인 아님).
- 제외한 위험/untracked 파일이 있으면 함께 표시한다.

승인 질문은 AskUserQuestion 으로 한다(승인 / 메시지 수정 / 파일 조정 / 취소). 승인 전에는 stage 도 commit 도 하지 않는다.

## 5. 커밋 실행

승인 후:

- 각 커밋 단위마다 명시 파일 경로만 stage: `git add -- <path> <path> ...`. `git add .` / `-A` 금지.
- `git commit -m "<subject>" -m "<body>"` (본문 없으면 `-m` 하나). 여러 줄 본문은 heredoc/`-F` 로 원문 유지.
- 분할이면 단위별로 stage→commit 을 순서대로 반복한다.
- pre-commit 훅을 존중한다. `--no-verify` / `--no-gpg-sign` 으로 우회하지 않는다. 훅이 실패하면 그 출력을 사용자에게 그대로 전하고 멈춘다 — 근본 원인을 알린다.

## 6. 검증

각 커밋 후 `git log -1 --stat`(분할이면 `git log -<N> --oneline`)로 결과를 확인해 사용자에게 보고한다. 커밋 후 예상치 못한 잔여 변경(`git status`)이 있으면 알린다.

## 금지사항

- 명시 요청 없이 커밋 생성.
- `git add .` / `git add -A`.
- Co-Authored-By 등 footer 부착.
- 훅 우회(`--no-verify`), 서명 우회(`--no-gpg-sign`).
- 본문에 설계/plan 문서 참조.
- push (요청받지 않는 한). 이 스킬은 로컬 커밋까지만 한다.
