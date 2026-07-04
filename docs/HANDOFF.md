# 핸드오프 — 다음 세션 시작점

## 한 줄 상태
설계 단계 **완료**. `docs/DESIGN.md` v2 커밋·푸시됨. 앱 코드는 아직 0줄.
다음: **라이브 프로브 스크립트(§7 실측)** 또는 **M1 스캐폴딩**.

## 먼저 읽을 것 (순서대로)
1. `CLAUDE.md` (자동 로드) — 코딩 가이드 + 커밋 규칙(§5)
2. `docs/DESIGN.md` — 전체 설계 스펙. 특히 결정 원장 §2, 쓰기/스케줄링 §5,
   API 검증 §7, v1 마일스톤 §10
3. 이 파일

## 확정된 것 (요약 — 상세는 DESIGN.md §2)
- **Windows 데스크톱 앱**, **Tauri + React + TypeScript**
- Gantt = **SVAR React Gantt** (`@svar-ui/react-gantt`, **MIT** — npm 실물 검증)
- **자체 호스팅 OpenProject APIv3**, **API 키** 인증(Rust 측 보관, 키 IPC write-only)
- **단일 프로젝트 Gantt + 프로젝트 피커**
- **v1 편집**: 드래그 일정 · 인라인 필드 · 계층 + **읽기전용 의존선** + 안전장치
  (reparent 확인 · 캐스케이드 이동 하이라이트 · 앱레벨 Undo)
- 온라인 우선 **인메모리 캐시** + 낙관적 쓰기 + **프로젝트 단위 직렬 쓰기 큐**
- UX: 모던·미니멀(Linear/Notion), 점진적 노출 + 색상/비색상 인코딩 + 검색·필터 + 사이드 peek

## 검증 완료 (12-에이전트 심층검증, npm 실물 근거)
큰 결정은 **전부 통과**. 실제 갭은 "미세 명세"였고 **DESIGN.md v2에 이미 반영**됨:
재조회 범위(§5.1) · 409/422 분리(§5.2) · 드래그 종류별 계약(§5.3) · 쓰기 큐·롤백(§5.4) ·
실패 상태(§6) · 페이지네이션 total 루프(§7) · 무료/PRO 무동작 함정(§8) · 비색상 단서·
접힘 지연노출(§9) · 한국어 로케일 자작(§13). 원본 검증 리포트는 세션 산출물이라 유실됨(요지는 문서에 반영).

## ⚠️ 다음 세션이 반드시 알아야 할 환경 제약
1. **인스턴스 주소 유실**: `docs/INSTANCE.local.md`(대상 OP 주소)는 **git-ignored**라 새 세션
   클론에 **없다**. 사용자에게 다시 물어라. (public 저장소라 커밋 금지 — 대신 앱 설정화면/로컬 노트로)
2. **egress 차단**: 이 환경은 `openproject.org`·`svar.dev`·대상 인스턴스에 **못 닿는다**
   (CONNECT 403 / 타임아웃). 공식 문서·인스턴스 라이브 검증 불가. **npm 레지스트리는 접근 가능**.
3. **라이브 검증(§7의 12항목)**: 사용자가 **본인 네트워크에서 API 키로** 프로브 스크립트를
   실행 → 출력만 회수해 확정한다. **API 키는 세션에 공유하지 않는다.**
4. **유용한 참조(npm으로 열람 가능)**: 공식 문서가 막혔으니 OP APIv3 사용 패턴은 실제 클라이언트
   패키지를 `npm pack` 해서 읽어 참고 — `opctl`, `openproject-mcp`, `n8n-nodes-open-project-api`
   (단, `openproject-mcp`는 단일 페이지만 가져오는 truncation 버그가 있으니 반면교사 — §7 참고).
5. (관측됨) `AskUserQuestion`·`Workflow` 권한 스트림이 MCP 재연결 중 **간헐 실패**했음 → 재시도로
   회복. 실패 시 텍스트 승인/재시도로 대응.

## 다음 할 일 (택1 또는 병행)
- [ ] **B — 라이브 프로브 스크립트** (추천 먼저): 단일 파일 `node probe.mjs`, `OP_URL`/`OP_TOKEN`
      env. §7 미해결 실측 — 캐스케이드 재조회 범위, `percentageDone` writable(/form), `pageSize`
      clamp, `scheduleManually` 동작, 부모 날짜 writable, 근무일, WBS 정렬, 필수/커스텀 필드.
- [ ] **A — M1 스캐폴딩**: `create-tauri-app`(React+TS) 뼈대 → 설정화면(OP URL+API키, write-only
      IPC, 자격증명 저장) → Rust `reqwest`로 `GET /api/v3/projects` 연결 확인.
      앞 두 단계는 인스턴스 없이 가능, 연결 확인만 실 인스턴스 필요.

## 작업 브랜치 / 커밋
- 브랜치: `claude/openproject-ux-ui-z1j3wc` (원격 동기됨)
- 커밋: Conventional Commits, 한국어 subject, **footer 없음**, `git add .` 금지, 승인 게이트
  (CLAUDE.md §5). 커밋 의도 보이면 `/commit` 스킬 사용.
