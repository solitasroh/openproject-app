# OpenProject Gantt 데스크톱 앱 — 설계 문서 (v2)

> OpenProject의 WP/Gantt UX를 사용자 편의적으로 개선하는 **별도 Windows 데스크톱 앱**.
> OpenProject REST APIv3로 데이터에 접근하고, 주 화면은 WBS 기반 WP를 추적/편집하는 Gantt.
>
> v2는 12-에이전트 심층 검증(다관점 사실검증 + 적대적 챌린지, npm 실물 근거)을 반영해
> 쓰기 경로·실패 상태·UX 안전장치를 보강한 판이다. 큰 결정(§2)은 모두 검증 통과했다.

## 1. 목표와 페인포인트

기존 OpenProject의 WP/Gantt UX가 불편하다는 문제의식에서 출발한다. 개선 우선순위:

1. **Gantt 편집 인터랙션** — 드래그 일정 변경, 리사이즈 기간 조정, 의존관계
2. **WBS 계층 탐색/조작** — 들여쓰기/내어쓰기, 부모-자식 이동, 대량 트리
3. **대규모 프로젝트 성능/탐색** — WP가 많을 때 로딩·스크롤·필터 접근성

핵심 가치: **많은 WP를 한눈에 조망(overview) + 쉽게 관리(manage)**.

## 2. 결정 원장

| # | 결정 | 값 | 비고 |
|---|------|-----|------|
| 1 | 형태 | Windows 데스크톱 앱 (신규 greenfield) | |
| 2 | 스택 | Tauri + React + TypeScript | |
| 3 | Gantt | SVAR React Gantt (`@svar-ui/react-gantt`) | **MIT** 확정, v1은 무료 코어로 충분(§8) |
| 4 | OP 연동 | 자체 호스팅 APIv3, 네이티브 HTTP | CORS 없음 |
| 5 | 인증 | API 키(개인 액세스 토큰) | Windows 자격증명 저장소, 키 IPC write-only(§3) |
| 6 | 범위 | 단일 프로젝트 Gantt + 프로젝트 피커 | |
| 7 | v1 편집 | 드래그 일정변경 · 인라인 필드편집 · 계층편집 + **읽기전용 의존선** | 의존관계 *편집*은 v1.1 |
| 8 | v1 안전장치 | reparent 확인 · 캐스케이드 이동 하이라이트 · 앱레벨 Undo | SVAR Undo는 PRO라 앱레벨 |
| 9 | UX 방향 | 모던·미니멀 (Linear/Notion 계열) | 키보드 중심, 밀도 조절 |
| 10 | 조망/관리 v1 | 점진적 노출 · 색상+비색상 인코딩 · 검색/필터+사이드패널 | 미니맵·그룹·저장뷰·대량편집은 v1.1+ |
| 11 | 캐싱/동기화 | 온라인 우선 + 인메모리 캐시 | 낙관적 쓰기 + 롤백 + 직렬 쓰기 큐(§5) |
| 12 | 사용자 | 소규모 내부 팀 (각자 설치) | 설정화면, 자동업데이트·서명은 v1.1 |
| 13 | UI 언어 | 한국어 우선 | 로케일 자작 필요(§13) |

**실무 디폴트**: 앱 크롬은 Tailwind + shadcn/ui, SVAR Gantt는 테마 맞춤 / API 호출은
Rust(reqwest) 측에서 수행해 API 키를 JS에 노출하지 않음.

**대상 인스턴스**: 자체 호스팅 OP. 저장소가 public 이므로 개발/검증 대상 인스턴스의 구체
주소는 커밋하지 않고 git-ignored 로컬 노트(`docs/INSTANCE.local.md`)에 보관하며, 런타임에는
설정 화면에서 OP 주소 + 개인 API 키를 입력한다. ⚠️ 인스턴스가 HTTP(비-TLS)면 Tauri/reqwest
에서 평문 HTTP 허용 + 신뢰 네트워크(LAN/VPN) 전제. API 키는 커밋 금지·Windows 자격증명
저장소에만 보관한다.

## 3. 아키텍처

```
[React/TS UI]  ── invoke/event ──  [Tauri Rust core]  ── HTTPS/HTTP ──  [OpenProject APIv3]
 · SVAR Gantt (트리그리드+타임라인)     · reqwest HTTP 클라이언트              · /projects
 · 검색/필터/사이드 상세패널            · API 키 = OS 자격증명 저장소            · /work_packages (+/form)
 · 인메모리 스토어 (React Query류)      · 프로젝트 단위 직렬 쓰기 큐            · /relations
```

- **API 호출은 Rust 측**에서: 키가 프런트 JS 번들·메모리에 노출되지 않고, CORS도 무관.
- **키 IPC는 write-only**: 설정에서 키 저장(set)만 허용, JS로 read-back/prefill 금지(profile-id
  참조). reqwest `Authorization` 헤더는 로그에서 redaction(디버그 타깃 노출 방지).
- **프런트는 인메모리 스토어**로 단일 프로젝트 WP 집합 보관(수백~수천 규모 충분). store는
  **프로젝트 키로 스코프**해 프로젝트 전환 시 오배송 PATCH를 방지한다.
- **미저장 편집 수명주기**: 낙관적 편집이 in-flight/미저장인 상태에서 프로젝트 전환·새로고침이
  일어나면 편집 유실·화면 되돌림·stale 409 위험 → 경고·보류·병합 규칙을 적용한다(§5). 주기
  새로고침은 v1에서 **수동 새로고침만** 확정하고 자동 폴링은 보류.
- SVAR RestDataProvider의 지연로드(request-data/provide-data)는 **의도적으로 미사용** —
  WBS 표시 번호 계산·필터 시 조상 유지·전역 검색이 전량 로드를 강제하기 때문. 대신 §7의
  페이지 루프로 프로젝트 WP를 전량 적재한다.

## 4. 데이터 매핑 (OP WorkPackage ↔ SVAR)

| SVAR `ITask` | OpenProject WP | 비고 |
|--------------|----------------|------|
| `id` | `id` | |
| `text` | `subject` | |
| `start` | `startDate` (부모는 `derivedStartDate`) | date-only |
| `end` | `dueDate` (부모는 `derivedDueDate`) | date-only, 없으면 막대 없는 행(§8) |
| `progress` | `percentageDone` | 0–100, 계산 모드 주의(§7) |
| `parent` | `_links.parent` | 계층(관계 아님) |
| `type` | 파생 | 자식 있으면 `summary`, milestone 타입이면 `milestone`, 그 외 `task` |
| `$custom` | `lockVersion` 등 OP 고유 필드 | `ITask[key:string]:any`에 적재 |

- **`ILink{source,target,type}`** ↔ OP 관계: `precedes`(FS) → `e2s`, `follows`는 from/to
  반전, `lag`(delay, 일수)는 `ILink` 확장 필드에 보관. **v1은 읽기전용 렌더**, 편집은 v1.1.
- **type 파생 주의**: SVAR 무료 코어는 summary 자동관리를 하지 않는다 → 어댑터가 "자식 있으면
  summary" 로 계산하고 날짜는 OP derived 값을 사용. **reparent(M5) 시 옛/새 부모의 type을
  재계산**한다.
- **커스텀 필드**: v1은 내장 필드만 매핑하고 커스텀 필드는 명시적 제외(필요 시 /form 스키마
  기반 후속). 타입별 required 필드는 /form이 권위.

## 5. 쓰기/스케줄링 원칙 (핵심)

> **OpenProject 서버가 스케줄링의 유일한 진실 원천이다.** 우리 쪽 스케줄링 엔진 없음
> → SVAR PRO의 자동 스케줄링/서머리 자동화 불필요.

기본 루프: SVAR가 사용자 *의도* 포착 → (필요 시) `/form` 선검증 → `lockVersion` 담아 PATCH
→ **서버 재계산 결과를 정답으로 받아** 인메모리 스토어 조정.

### 5.1 재조회 범위 (조상 stale 방지)
리프의 날짜를 바꾸면 사용자가 관계를 만들지 않아도 **조상의 `derivedStartDate/derivedDueDate`
가 항상 바뀐다**. 따라서 순수 subtree(자기+후손)만 재조회하면 접힌 조상 요약막대가 stale로
남는다. → PATCH 후 **"자기 + 모든 조상 + `updatedAt ≥ 요청시각`인 WP"** 를 재조회한다
(updatedAt 필터 컬렉션 질의). 조상/영향 WP를 PATCH 응답 임베드로 회수할지 별도 GET으로 할지는
**M1 검증 항목**(§7).

### 5.2 충돌·검증 실패 (409 vs 422 분리)
- **409 (lockVersion 충돌)**: 재조회 후 **사용자가 바꾼 필드만** 새 lockVersion으로 재-PATCH
  (부분 병합 → 타인의 다른-필드 변경 보존), 같은-필드 충돌만 충돌 UI. **blind 전체 스냅샷
  재전송 금지.**
- **422 (검증 실패)**: 필드별 오류 표시, **재시도 금지**. 예: 허용 안 되는 상태전이,
  work-based 모드의 `percentageDone` 직접편집.

### 5.3 드래그 분기 (WP 종류별)
SVAR `ITask`에는 `scheduleManually` 개념이 없어 앱이 계약을 전적으로 설계한다.
- **리프(수동)**: 이동/리사이즈 → `startDate`/`dueDate` PATCH. 드래그로 지정한 날짜를
  고정하려면 대상 WP를 **`scheduleManually=true`로 명시**.
- **자동스케줄(`scheduleManually=false`)**: 드래그해도 서버가 스냅백 → 편집 억제. SVAR엔
  per-task 드래그 비활성 플래그가 없어 `readonly` 또는 `api.intercept`로 처리.
- **요약(부모)막대**: 무료 코어에서도 드래그 가능(자식을 dx만큼 이동하는 캐스케이드). "요약
  드래그=부모 파생날짜 PATCH"는 틀림 → emit되는 자식 이동들을 개별 PATCH로 변환하거나 억제.

### 5.4 쓰기 큐 · 롤백
- **롤백**: PATCH 실패 시 복원할 **편집전 스냅샷** 보관.
- **직렬화**: Rust 측 **프로젝트 단위 직렬 쓰기 큐**(A의 연쇄 재조회 중 B 편집이 stale
  lockVersion으로 덮이는 순서 위험 차단). SVAR RestDataProvider의 배치 큐는 `init(api)`
  인터셉트로 우회되므로 앱이 직렬화를 담당.
- **coalesce**: 드래그 중간 이벤트는 `inProgress`로 합치고 **최종 이벤트만 PATCH**.

## 6. 실패 상태 / 에러 처리

100% 원격 의존 앱이므로 실패 경로를 1급으로 설계한다.

| 상태 | UX |
|------|-----|
| 로딩 | 프로젝트 로드 진행표시(페이지 진척), 스켈레톤 |
| 빈 프로젝트 | 빈 상태 안내 + WP 생성 유도 |
| 401/403 (인증/권한) | 명확한 오류 + 설정(키) 재입력 경로. 내 키 권한 밖 필드는 비활성 |
| 5xx / 타임아웃 | 재시도 가능한 오류 배너, 낙관적 편집은 롤백(§5.4) |
| 네트워크 단절 | 오프라인 배지, 쓰기 비활성(오프라인 편집은 v1.1) |
| 부분 페이지 로드 실패 | 실패를 **표면화**(조용한 truncation 금지, §7) + 부분 재시도 |

각 마일스톤 게이트에 **실패 어서션**을 포함한다(§10).

## 7. OpenProject API — 확신 · 라이브 검증 항목

**확신도 높음** (독립 클라이언트 3종 동작과 수렴 일치):
`PATCH /work_packages/{id}` + `lockVersion`(stale→409) · `/form` 선검증·허용값 ·
관계 `precedes/follows`+`lag`·계층 `_links.parent` · date-only 날짜 ·
`POST /projects/{id}/work_packages` 생성 · PAT 인증 · 계층/상태/담당자가 컬렉션에 임베드(N+1 없음).

**페이지네이션 파이프라인**: `GET /work_packages`의 pageSize는 인스턴스 설정
(`api_max_page_size`, 100 초과 가능)이고 **초과 요청을 서버가 조용히 clamp**하며 offset은
1-base 페이지번호다. → **`collection.total` 기반 페이지 루프**로 전량 적재, 점진 렌더 vs 일괄
대기·부분실패 처리 명시(조용한 truncation 금지). 정적 OpenAPI 스펙은 필드/타입만 선확인 가능,
`writable`/`required` 플래그는 `/form`·라이브 인스턴스에서만 확정.

**M1에서 실 인스턴스로 검증** (대상: `docs/INSTANCE.local.md`의 인스턴스):
- 진행률 계산 모드(work/status-based)에서 `percentageDone.writable` 여부 — /form schema로 편집기 분기
- **쓰기 후 서버 캐스케이드 실제 범위** — 재계산·이동되는 WP 집합, PATCH 응답 포함 여부 (가장 깊은 리스크)
- `api_max_page_size`(100 초과 가능 여부)와 초과 요청 clamp 동작
- `scheduleManually` 기본값(버전별)과 자동 막대 드래그 시 스냅백 동작
- 자식 있는 부모의 start/end 읽기전용 여부(/form writable) — 부모 막대 편집 가능성
- 근무일 설정 + per-WP `ignoreNonWorkingDays`의 duration 영향
- OP 웹 수동 정렬을 `/queries`로 읽어 WBS 행순서와 일치시킬 수 있는지
- 타입별 required 필드 + 커스텀 필드 요구(/form 권위)
- select/embed 지원 범위(페이로드 축소·관계 N+1 완화)
- 부모 `percentageDone`가 자식 집계를 API로 반환하는지(요약막대 진행률 소스)
- Basic 인증 활성 여부(일부 인스턴스 비활성 가능)
- 대상 하드웨어에서 ~1만 태스크 가상화 성능(M2 벤치)

> ⚠️ 이 개발 세션의 네트워크 정책은 대상 인스턴스에 도달하지 못한다. 라이브 검증은 사용자
> 네트워크에서 실행하는 **독립 프로브 스크립트**(API 키 로컬 실행, 출력만 회수) 또는 앱 M1로
> 수행한다.

## 8. SVAR React Gantt — 무료 코어 vs PRO 경계

**무료(MIT) 코어 — v1 커버** (npm pack 실물 확인):
트리그리드+요약막대 · 드래그 이동/리사이즈 · 인라인 그리드 편집(컬럼 `editor`) · 진행률 막대 ·
의존선 **시각화** · 필터/정렬 · 줌/타임스케일 · **가상화(~1만)** · `taskTemplate`(막대 커스텀) ·
`highlightTime`(컬럼 틴트) · `init(api)` 인터셉트 · reparent(indent/move-task) · 로컬라이제이션
프레임워크.

**PRO(유료) — v1 불필요**:
크리티컬 패스 · 슬랙 · 베이스라인 · 그룹/스윔레인 · 자동 스케줄링 · 서머리 자동화 · **롤업 마커** ·
WBS 코드 · Undo/Redo · Export(PNG/PDF/Excel/MSProject) · 리소스 계획/워크로드 ·
work-time/per-task 캘린더 · **unscheduled tasks** · split tasks · **vertical markers**.

**함정(코드에 고정)**:
- 무료 빌드의 `.d.ts`/JS가 PRO 설정·액션(criticalPath·undo·baselines·rollups·groupBy·schedule·
  export 등)을 **그대로 노출** → 무료 빌드에서 컴파일되지만 런타임 미번들로 **조용히 무동작**
  (TS가 못 잡음). → **무료 코어 액션 화이트리스트**를 앱에 고정.
- **오늘선**: Vertical markers=PRO → 무료 `highlightTime`(오늘 컬럼 틴트)로 구현.
- **날짜없는 WP**: Unscheduled tasks=PRO → 무료 코어에선 **막대 없는 그리드행**으로 렌더.
- **롤업 마커=PRO** → 접힘 지연 노출은 taskTemplate 파생으로(§9).

## 9. UX 설계 — 조망 + 관리

**조망 (많아도 한눈에)**
- 점진적 노출: 기본 레벨 1~2만 펼침, 접으면 자식 범위를 아우르는 요약막대.
- 접힘 지연 노출: 요약막대에 **"하위 최악 상태" 뱃지/테두리**(taskTemplate 파생 — 롤업이 PRO라
  앱 파생값으로). 접힌 부모 안 overdue 리프가 숨지 않게 함.
- 색상 인코딩 + **비색상 단서**(아이콘/패턴/텍스트/테두리): 색만으로 상태 구분은 WCAG 1.4.1
  위반 + CVD/밀도에서 salience 저하 → taskTemplate로 병기. **salience 팔레트**(예외 상태만
  고채도). 막대 색은 **상태색 vs 담당자색 배타 토글**.
- 오늘선(highlightTime), 줌 + Zoom-to-fit, 가상화. (미니맵은 v1.1)

**관리 (쉽게 조작)**
- 검색/필터 1급: 항상 보이는 검색·필터칩, 필터 시 매칭 WP의 상위 계층 유지.
- **overdue 상시 필터칩**(파생술어) + **연체일 정렬** + 매칭 카운트 — 핵심 질의를 1급으로.
- 사이드 상세 패널: 전체화면 점거 대신 옆 패널 peek(overlay/도킹·핀 시맨틱 명시).
- 키보드 우선 + 아웃라이너식 추가(Tab/Enter=자식/형제, 들여쓰기=부모변경). **모달 소유권 규칙:
  에디터 열림 시 Tab/Enter는 에디터가 점유**(구조편집과 이중점유 해소) + 키맵 치트시트.
- reparent·드래그는 서버 캐스케이드를 유발 → **확인 다이얼로그 + 이동된 막대 하이라이트 + 앱
  Undo**(§10 안전장치)로 "유령 이동"을 설명.
- 밀도 토글(comfortable/compact).

## 10. v1 범위와 마일스톤

각 마일스톤 게이트는 **성공 + 실패** 어서션을 포함한다(CLAUDE.md §4 falsifiable).
순수 함수(어댑터·§5 쓰기 로직·WBS 번호·페이지네이션 루프)는 **픽스처 단위테스트**로 커버.

1. **셸+연결** — Tauri 뼈대, 설정화면(OP URL+API키, write-only IPC), 자격증명 저장, 연결 확인
   · 성공: `/projects` 200, 목록 표시 · 실패: 잘못된 URL/키→명확한 오류
2. **읽기 Gantt** — 프로젝트 피커 → WBS 트리그리드+타임라인, 깊이별 접기+요약막대, 줌, 색상+
   비색상 인코딩, 오늘선(highlightTime), 가상화, **읽기전용 의존선**
   · 성공: 목표 WP 수 초기 렌더 < X초, 스크롤 ≥ 30fps, 계층·WBS 번호·형제순서 OP 일치
   · 실패: 부분 페이지 로드 실패가 사용자에게 표면화, 로딩 진행표시
3. **인라인 편집** — 제목·상태·담당자·날짜·진행률, `/form` 검증, 낙관적 업데이트, 409/422 분리
   · 검증: 편집 서버 반영, 충돌(409)·검증실패(422) 각각 올바른 UX, 실패 시 롤백
4. **드래그 편집** — 막대 이동/리사이즈 → 날짜 PATCH → 재조회(자기+조상+updatedAt), 캐스케이드
   이동 하이라이트, 앱 Undo
   · 전제: (의존선 검증용) precedes 관계는 OP 웹에서 사전 생성
   · 검증: 드래그 결과가 OP와 일치, 관계로 밀린 WP·조상 요약막대 stale 없음
5. **계층 편집** — 하위/형제 추가, 들여쓰기/내어쓰기(부모변경, 확인 다이얼로그), 삭제, 부모 type
   재계산
   · 검증: WBS 변경이 OP 트리와 일치, reparent 캐스케이드 반영
6. **검색/필터/사이드패널 마감** — overdue 칩·연체 정렬·매칭 카운트, peek 패널
   · 검증: 필터 시 상위 계층 유지, 상세 peek 동작

## 11. v1.1+ 백로그

의존관계 **편집**(드래그 연결/삭제) · SQLite 영속 캐시/오프라인 읽기 · 미니맵 개요 ·
그룹/스윔레인 · 저장된 뷰 · 대량 편집 · 자동 새로고침/폴링 · 자동 업데이트·코드 서명 ·
OAuth2(PKCE) · (PRO 검토) 크리티컬 패스·베이스라인·Export·Undo/Redo.

## 12. 리스크

- **OP 스케줄링 캐스케이드 범위** (가장 깊음): 드래그가 서버 다중 재계산 유발 → §5.1 재조회
  전략 + M1 라이브 검증으로 확정. 미확정 시 §7 프로브 스크립트로 선실측.
- **진행률 계산 모드**: work-based면 `percentageDone` 직접편집 불가 가능 → /form writable로 분기(M3).
- **무료/PRO 무동작 함정**: PRO 액션이 조용히 무동작 → 화이트리스트 고정(§8).
- **한국어 로케일 자작 비용**(§13).

## 13. i18n / UI 언어

`gantt-locales`는 en/cn만, `core-locales`는 ja는 있으나 **ko 미제공** → 한국어는 무료
로컬라이제이션 프레임워크로 **자작**한다. i18n 범위는 3층으로 분리: **① SVAR Gantt 위젯 문자열**
(로케일 자작) · **② shadcn 앱 크롬**(자체 문자열) · **③ 서버 반환 라벨**(OP 인스턴스 언어에
종속 — 한국어 인스턴스면 한국어로 옴). 날짜/숫자 포맷은 로케일 규칙 적용.
