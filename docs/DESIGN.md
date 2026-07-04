# OpenProject Gantt 데스크톱 앱 — 설계 문서

> OpenProject의 WP/Gantt UX를 사용자 편의적으로 개선하는 **별도 Windows 데스크톱 앱**.
> OpenProject REST APIv3로 데이터에 접근하고, 주 화면은 WBS 기반 WP를 추적/편집하는 Gantt.

## 1. 목표와 페인포인트

기존 OpenProject의 WP/Gantt UX가 불편하다는 문제의식에서 출발한다. 개선 우선순위로
확정한 세 가지 페인포인트:

1. **Gantt 편집 인터랙션** — 드래그 일정 변경, 리사이즈 기간 조정, 의존관계 연결이 번거로움
2. **WBS 계층 탐색/조작** — 들여쓰기/내어쓰기, 부모-자식 이동, 대량 트리 다루기
3. **대규모 프로젝트 성능/탐색** — WP가 많을 때 로딩·스크롤이 느리고 필터·그룹이 메뉴에 숨어 있음

핵심 가치: **많은 WP를 한눈에 조망(overview) + 쉽게 관리(manage)**.

## 2. 결정 원장

| # | 결정 | 값 | 비고 |
|---|------|-----|------|
| 1 | 형태 | Windows 데스크톱 앱 (신규 greenfield) | |
| 2 | 스택 | Tauri + React + TypeScript | |
| 3 | Gantt | SVAR React Gantt (`@svar-ui/react-gantt`) | **MIT** 검증 완료, v1은 무료 코어로 충분 |
| 4 | OP 연동 | 자체 호스팅 APIv3, 네이티브 HTTP | CORS 없음 |
| 5 | 인증 | API 키(개인 액세스 토큰) | Windows 자격증명 저장소 보관 |
| 6 | 범위 | 단일 프로젝트 Gantt + 프로젝트 피커 | 프로젝트 전환 가능 |
| 7 | v1 편집 | 드래그 일정변경 · 인라인 필드편집 · 계층편집 | 의존관계 편집은 v1.1 |
| 8 | UX 방향 | 모던·미니멀 (Linear/Notion 계열) | 키보드 중심, 밀도 조절 |
| 9 | 조망/관리 v1 | 점진적 노출(깊이별 접기+요약막대) · 색상 인코딩+오늘선/지연 · 검색/필터+사이드패널 | 미니맵·그룹·저장뷰·대량편집은 v1.1+ |
| 10 | 캐싱/동기화 | 온라인 우선 + 인메모리 캐시 | 낙관적 쓰기 + lockVersion 충돌처리 |
| 11 | 사용자 | 소규모 내부 팀 (각자 설치) | 설정화면 필요, 자동업데이트·서명은 v1.1 |
| 12 | UI 언어 | 한국어 우선 | SVAR 로컬라이제이션 활용 |

**실무 디폴트**: 앱 크롬은 Tailwind + shadcn/ui, SVAR Gantt는 테마 맞춤 / API 호출은
Rust(reqwest) 측에서 수행해 API 키를 JS에 노출하지 않음.

## 3. 아키텍처

```
[React/TS UI]  ── invoke/event ──  [Tauri Rust core]  ── HTTPS ──  [OpenProject APIv3]
 · SVAR Gantt (트리그리드+타임라인)     · reqwest HTTP 클라이언트           · /projects
 · 검색/필터/사이드 상세패널            · API 키 = OS 자격증명 저장소          · /work_packages (+/form)
 · 인메모리 스토어 (React Query류)      · (v1.1) SQLite 캐시                  · /relations
```

- **API 호출은 Rust 측**에서: API 키가 프런트 JS 번들·메모리에 노출되지 않고, CORS도 무관.
- **프런트는 인메모리 스토어**로 단일 프로젝트 WP 집합을 보관(수백~수천 규모는 충분).
- 외부 변경은 수동/주기 새로고침으로 반영(실시간 협업은 비목표).

## 4. 데이터 매핑 (OP WorkPackage ↔ SVAR)

| SVAR `ITask` | OpenProject WP | 비고 |
|--------------|----------------|------|
| `id` | `id` | |
| `text` | `subject` | |
| `start` | `startDate` (부모는 `derivedStartDate`) | date-only |
| `end` | `dueDate` (부모는 `derivedDueDate`) | date-only |
| `progress` | `percentageDone` | 0–100 (계산 모드 주의, §6) |
| `parent` | `_links.parent` | 계층 |
| `type` | 파생 | 자식 있으면 `summary`, milestone 타입이면 `milestone`, 그 외 `task` |

`ILink{source,target,type}` ↔ OP `precedes`/`follows` 관계 + `lag`(delay). **v1.1**에서 사용.

## 5. 쓰기/스케줄링 설계 원칙 (핵심)

> **OpenProject 서버가 스케줄링의 유일한 진실 원천이다.**

1. SVAR Gantt는 사용자 *의도*(새 날짜/부모/진행률)만 포착한다.
2. 필요 시 `POST /work_packages/{id}/form`으로 **선검증** 및 허용값 조회.
3. `lockVersion`을 담아 `PATCH /work_packages/{id}`.
4. **서버가 재계산한 결과(관계로 밀려난 다른 WP 포함)를 정답으로 받아** 인메모리 스토어를
   조정한다. 영향받은 부분은 재조회한다.
5. `lockVersion` 충돌(HTTP 409) 시 → 재조회 후 재시도하거나 충돌을 사용자에게 표시.

이 원칙으로 **부모 날짜 파생·의존 재계산·근무일 계산을 전부 서버에 위임**한다. 따라서 SVAR
PRO의 자동 스케줄링/서머리 자동화가 **불필요**하다. 드래그 후에는 영향 subtree를 재조회해
서버 재계산을 화면에 반영한다.

## 6. OpenProject API — 확신/검증 항목

**확신도 높음** (제 지식 기준):
- `PATCH /api/v3/work_packages/{id}` + `lockVersion` 필수, stale이면 **409 Conflict**.
- `/work_packages/{id}/form` (및 생성 폼)으로 검증·허용값(상태전이·담당자·타입) 조회.
- 생성: `POST /projects/{id}/work_packages`, 필수 = subject + type(+project).
- 관계: `/api/v3/relations`, Gantt에 유효한 타입 `precedes`/`follows` + `lag`. 계층은
  관계가 아니라 `_links.parent`.
- 날짜 `startDate`/`dueDate`는 date-only(YYYY-MM-DD). 부모는 `derivedStartDate/DueDate`.
- `scheduleManually`(bool): true=수동(날짜 고정), false=자동(관계/자식이 날짜 구동).

**M1에서 실 인스턴스로 반드시 검증**:
- 진행률 계산 모드 — work-based면 `percentageDone`를 직접 못 쓰고 work/remaining에서 파생될 수 있음.
- `GET /work_packages` 최대 `pageSize`(인스턴스 설정, 보통 100~200)와 필터 문법.
- WBS 정렬/순서 표현 방식(전역 position 필드 부재 → 표시용 WBS 번호는 클라이언트 계산).
- 근무일 설정과 per-WP `ignoreNonWorkingDays`가 duration에 미치는 영향.
- 페이로드 축소를 위한 select/embed 지원 범위, 관계 N+1 완화 방법.

> ⚠️ 이 세션의 네트워크 정책이 openproject.org 문서를 차단하여 공식 문서 정밀 확인은 보류.
> 실 인스턴스가 최종 진실 원천이므로 M1에서 실증한다.

## 7. SVAR React Gantt — 무료 코어 vs PRO 경계

**무료(MIT) 코어 — v1 전 범위 커버**:
트리그리드+요약막대 · 타임라인 드래그 · 인라인 그리드 편집(컬럼 `editor`) · 진행률 막대 ·
필터 · 정렬 · 줌/타임스케일(시간~스프린트) · **1만 개 가상화** · 로컬라이제이션 ·
`taskTemplate`로 막대 완전 커스텀(색상 인코딩·오늘/지연 강조) · `init(api)` 액션 인터셉트
(백엔드 동기화) · 툴바/컨텍스트메뉴/툴팁.

**PRO(유료) — v1 불필요, 필요 시 라이선스 검토**:
크리티컬 패스 · 슬랙 · 베이스라인 · 그룹/스윔레인 · 자동 스케줄링 · 서머리 자동화 ·
롤업 마커 · WBS 코드 · Undo/Redo · Export(PNG/PDF/Excel/**MS Project**) · 리소스 계획.

## 8. UX 설계 — 조망 + 관리

**조망 (많아도 한눈에)**
- 점진적 노출: 기본 레벨 1~2만 펼침, 접으면 자식 범위를 아우르는 요약막대(진행률 포함).
- 색상 인코딩: 상태/담당자별 막대 색, `overdue=빨강`·`done=회색`, 오늘 세로선.
- 줌 + Zoom-to-fit. (미니맵은 v1.1)
- 가상화로 수천 개도 부드럽게.

**관리 (쉽게 조작)**
- 검색/필터 1급: 항상 보이는 검색·필터칩, 필터 시 매칭 WP의 상위 계층은 유지.
- 사이드 상세 패널: 클릭 시 전체화면 점거 대신 옆 패널 peek.
- 키보드 우선 + 아웃라이너식 추가(Tab/Enter로 자식/형제, 들여쓰기=부모변경).
- 밀도 토글(comfortable/compact).

## 9. v1 범위와 마일스톤

1. **셸+연결** — Tauri 뼈대, 설정화면(OP URL+API키), 자격증명 저장, 연결 확인
   · 검증: `/api/v3/projects` 200, 프로젝트 목록 표시
2. **읽기 Gantt** — 프로젝트 피커 → WBS 트리그리드+타임라인, 깊이별 접기+요약막대, 줌,
   색상/오늘선, 가상화
   · 검증: 수천 WP 조망이 부드럽고 계층이 OP와 일치
3. **인라인 편집** — 제목·상태·담당자·날짜·진행률, `/form` 검증, 낙관적 업데이트+409 처리
   · 검증: 편집이 서버 반영, 충돌 감지
4. **드래그 편집** — 막대 이동/리사이즈 → 날짜 PATCH → 서버 재계산 반영
   · 검증: 드래그 결과가 OP와 일치, 관계로 밀린 WP도 반영
5. **계층 편집** — 하위/형제 추가, 들여쓰기/내어쓰기(부모변경), 삭제
   · 검증: WBS 변경이 OP 트리와 일치
6. **검색/필터/사이드패널 마감**
   · 검증: 필터 시 상위 계층 유지, 상세 peek 동작

## 10. v1.1+ 백로그

의존관계 편집(드래그 연결) · SQLite 영속 캐시/오프라인 읽기 · 미니맵 개요 · 그룹/스윔레인 ·
저장된 뷰 · 대량 편집 · 자동 업데이트·코드 서명 · OAuth2(PKCE) · (PRO 검토) 크리티컬
패스·베이스라인·Export·Undo/Redo.

## 11. 리스크

- **OP 스케줄링 상호작용**: 드래그가 서버 다중 재계산을 유발 → M1~M4에서 실증하며 재조회
  전략 확정. (가장 깊은 기술 리스크)
- **진행률 계산 모드**: work-based면 진행률 직접 편집 불가 가능 → M3에서 분기 처리.
- **SVAR PRO 유혹**: 후반 기능(크리티컬 패스 등)에서 라이선스 결정 필요.
- **네트워크 정책**: 공식 문서 접근 차단 상태 → 실 인스턴스 실증에 의존.
