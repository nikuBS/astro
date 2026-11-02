# 기술 설계 문서 v0.1 — 레트로 게임 모바일 청첩장

- 상태: 초안 (W1-2 리뷰용, 11/12 1차 리뷰 → 11/14 확정)
- 대상: 프론트 2 · 백엔드 3 · 퍼블 1 · 기획 2
- 기준 자료: 기획서 덱, 투자자 데모(`wedding-quest-demo.html`), 기존 저장소 분석 보고서
- 범위: 베타(12/28 클로즈드) 기준. 레이싱 이후 장르는 확장 지점만 정의

---

## 0. 결정 요약

| 항목 | 결정(안) | 이유 |
|---|---|---|
| 웹 프레임워크 | 기존 저장소의 Astro 1.5 → **Astro 5**, `output: 'server'` (필요 페이지만 SSR) | 저장소 유지 결정, 청첩장별 OG 태그는 서버 렌더링이 필수 |
| UI 섬(island) | **React** (에디터), 게임은 프레임워크 없는 TS 모듈 | 에디터는 상태가 복잡, 게임은 번들 크기와 성능 우선 |
| 백엔드 | **Supabase** (Auth · Postgres · Storage · RLS · Edge Functions) | 백엔드 3명이 인프라 대신 도메인 로직에 집중 |
| 배포 | **Vercel** (대안: Cloudflare Pages) | Astro SSR 어댑터 성숙, 프리뷰 배포로 QA 편함 |
| 게임 렌더링 | Canvas 2D, 논리 해상도 **160×220** 고정 + 정수배 확대 | 데모에서 검증, 저사양에서도 가벼움, 픽셀 느낌 유지 |
| 게임 엔진 | 외부 엔진 없이 **자체 공통 런타임** (데모 코드 기반) | 15초 게임에 Phaser 등은 과함(번들 수백 KB) |
| 로그인 | 카카오 로그인 (Supabase OAuth) | 타깃 사용자, 카톡 공유와 같은 앱 키 |
| 결제 | 국내 PG(포트원 경유 검토) → 웹훅으로 플랜 반영 | 카드·간편결제 한 번에, 심사 병행 |

---

## 1. 시스템 구성

```
[신랑·신부 브라우저]                         [하객: 카톡 인앱 브라우저]
  /studio (에디터, React island)               /i/{slug}  (SSR: OG 태그 + 초기 데이터)
     │  ▲ 미리보기: iframe + postMessage           │
     ▼  │                                          ▼
 ┌──────────── Astro 5 (Vercel) ─────────────────────────┐
 │ SSR 페이지 · API 라우트(/api/*) · OG 이미지(/og/{slug}.png) │
 └──────┬─────────────────────────────┬─────────────────┘
        ▼                             ▼
  Supabase Postgres (RLS)       Supabase Storage (사진: 원본 비공개, 리사이즈본 공개 URL 서명)
        ▲
  PG 웹훅 → Edge Function(결제 검증 → plan/expires_at 갱신)
  스케줄러(Cron) → 만료 처리·파기
```

- 하객 페이지는 **로그인 없이** 열린다. 데이터는 공개 가능한 필드만 담은 "발행 스냅샷"에서 읽는다.
- 게임 코드는 하객 페이지에서 **장르 모듈만 지연 로드**한다 (`import('/games/rpg.js')`).

---

## 2. 저장소 구조 (기존 저장소 재편)

```
src/
  pages/
    index.astro              # 서비스 랜딩 (기존 청첩장 페이지는 /legacy 로 이동 후 삭제 예정)
    studio/[id].astro        # 에디터 (로그인 필요)
    i/[slug].astro           # 하객 청첩장 (SSR)
    og/[slug].png.ts         # 카톡 미리보기 이미지
    api/                     # publish, upload-url, guestbook, payment-webhook 등
  game/
    runtime/                 # 공통 런타임: loop, input, fx, audio, result
    genres/rpg/ artillery/   # 장르 모듈 (racing/ 은 오픈 베타)
    sprites/                 # 픽셀 스프라이트 데이터 + 팔레트
  scenes/                    # 타이틀·대화·보상·엔딩 (장르 공통 씬)
  editor/                    # React 에디터 컴포넌트
  lib/schema/                # InvitationConfig 타입 + zod 검증 (프론트·서버 공용)
supabase/
  migrations/  functions/
```

- `docs/` 수동 빌드 산출물과 GitHub Pages 배포는 **폐기**한다 (선행 정리 항목).

---

## 3. 데이터 모델

### 3.1 InvitationConfig (편집 상태 = 발행 데이터의 원본)

```ts
// src/lib/schema/invitation.ts  — zod 스키마에서 타입을 뽑아 프론트·서버가 같이 쓴다
type GenreId = 'rpg' | 'artillery' | 'racing';

interface InvitationConfig {
  schemaVersion: 1;
  genre: { id: GenreId; version: number };      // 장르 버전 고정: 게임 수정이 발행된 청첩장을 깨지 않게
  couple: {
    groom: { name: string; look: Look };
    bride: { name: string; look: Look };
    roleSwap?: boolean;                          // 구출하는 쪽/구출되는 쪽 반전
  };
  event: {
    datetime: string;                            // ISO, Asia/Seoul
    hall: string; hallDetail?: string;
    address: string;
    geo?: { lat: number; lng: number };
  };
  slots: {
    prologue: string[];                          // 장르가 정의한 개수만큼, 빈 값이면 기본 대사
    ending?: string;
    bossName?: string;
  };
  photos: PhotoRef[];                            // 순서 = 갤러리 순서, [0] = 합동기 컷인
  payment?: { groomLink?: string; brideLink?: string };  // 카카오페이 송금 링크만. 계좌번호는 베타 범위 밖
  options: { guestbook: boolean; music: boolean };
}
interface Look { hair: string; outfit: string }  // 팔레트 인덱스 문자열, 자유 색상은 받지 않음
interface PhotoRef { id: string; w: number; h: number }
```

- 슬롯 개수·글자 수 제한은 **장르 매니페스트**(5.3)가 정하고 서버에서 zod로 다시 검증한다.
- `genre.version`: 발행 시점의 장르 버전을 고정한다. 장르 코드는 `/games/rpg@1.js`처럼 버전별로 배포한다.

### 3.2 테이블

| 테이블 | 주요 컬럼 | 비고 |
|---|---|---|
| `profiles` | id(=auth.uid), kakao_id, created_at | |
| `invitations` | id, owner_id, slug(unique), draft jsonb, status(draft/published/expired/deleted), plan(free/p90/p180), expires_at, published_at | draft = InvitationConfig |
| `published_snapshots` | invitation_id, version, data jsonb, created_at | 하객이 읽는 공개 데이터. 발행할 때마다 새 버전 |
| `photos` | id, invitation_id, path_original, path_web, w, h, order | 원본은 비공개 버킷 |
| `guestbook` | id, invitation_id, nickname, message, clear_record, created_at, hidden | 신고·숨김 |
| `orders` | id, invitation_id, plan, amount, pg_tx_id, status, paid_at | PG 웹훅으로만 갱신 |
| `reports` | id, invitation_id, reason, created_at | 신고 |

### 3.3 권한 (RLS 초안)

- `invitations`, `photos`, `orders`: **소유자만** 읽기·쓰기 (`owner_id = auth.uid()`).
- `published_snapshots`: 익명 읽기 허용, 단 `invitations.status = 'published' and expires_at > now()`인 것만.
- `guestbook`: 익명 **삽입**은 API 라우트(레이트 리밋)를 통해서만, 읽기는 공개 스냅샷과 같은 조건. 소유자는 숨김 가능.
- `orders`: 클라이언트 쓰기 금지. Edge Function(service role)만 쓴다.

---

## 4. 주요 흐름

### 4.1 편집과 미리보기
1. 에디터는 `draft`를 로컬 상태로 들고 **1초 디바운스 자동 저장**(`PATCH invitations.draft`).
2. 미리보기는 하객 페이지와 **같은 번들**을 iframe(`/i/preview?mode=draft`)으로 띄우고, 에디터가 `postMessage({type:'config', config})`로 밀어 넣는다. → 데모의 "왼쪽 수정 → 오른쪽 즉시 반영"과 동일.
3. 하객 화면 코드가 하나라서 "미리보기와 실제가 다르다"는 문제가 원천적으로 없다.

### 4.2 사진 업로드
1. 클라이언트에서 긴 변 **2048px로 리사이즈 + EXIF(위치 정보) 제거** 후 업로드.
2. `/api/upload-url`이 Storage 서명 업로드 URL 발급 (플랜별 장수 제한을 여기서 검사).
3. 웹용(1080px, WebP)과 컷인용(320px)을 Edge Function에서 생성.

### 4.3 발행
1. `/api/publish`: draft를 zod + 장르 매니페스트로 검증 → `published_snapshots`에 새 버전 저장 → `status='published'`, 무료면 `expires_at = 예식일 + 30일`.
2. slug는 추측하기 어려운 8자 이상 랜덤(`/i/k7Qm2xPa`). 이름이 들어간 URL은 쓰지 않는다.
3. OG 이미지 캐시 무효화.

### 4.4 하객 열람
1. `/i/{slug}` SSR: 최신 스냅샷 조회 → `<meta og:*>` + 초기 데이터 인라인 → 정적 HTML 응답 (CDN 캐시 60초).
2. 클라이언트: 타이틀 씬 즉시 표시 → 장르 모듈 지연 로드 → 대화 → 게임 → 보상 → 엔딩.
3. **정보 보기**는 게임 로드와 무관하게 첫 화면부터 동작 (SSR HTML에 정보 카드가 이미 있음).

### 4.5 결제와 만료
- 결제창 → PG 웹훅 → Edge Function이 금액·주문 검증 → `orders.paid` + `invitations.plan/expires_at` 갱신. **클라이언트 콜백만으로는 절대 반영하지 않는다.**
- 매일 Cron: `expires_at` 지난 청첩장 `expired`로 전환(하객에게 "종료된 청첩장" 화면) → 30일 후 사진·스냅샷 파기, 이름 등 개인정보 마스킹.

### 4.6 카톡 공유
- 카카오 JS SDK `Kakao.Share.sendDefault` (피드 템플릿): 제목 = 장르 타이틀 문구, 이미지 = `/og/{slug}.png`.
- OG 이미지: Satori(+resvg)로 1200×630 생성. 픽셀 캐릭터 + 이름 + 날짜. 스냅샷 버전을 쿼리에 넣어 캐시 분리.

---

## 5. 게임 런타임 설계

데모(`wedding-quest-demo.html`)의 구조를 TS 모듈로 옮긴다.

### 5.1 공통 런타임이 맡는 것
- **루프**: `requestAnimationFrame`, dt 상한 50ms, 일시정지(정보 보기·탭 전환 시 `visibilitychange`).
- **입력**: Pointer Events를 논리 좌표(160×220)로 변환해 `down/move/up` 전달, 하단 컨트롤 영역 버튼은 `cmd(name)`.
- **연출 킷(fx)**: `shake(px)`, `hitStop(sec)`, `flash(color)`, `popText(x,y,text,color)`, `burst(x,y,color,n)`, `cutIn(photo|sprites, text)`.
- **오디오**: Web Audio로 효과음 합성 (`sfx('hit'|'crit'|'boom'|...)`). 첫 사용자 탭에서 AudioContext 재개, 음소거 토글.
- **결과**: `finish({ win, stars })` → 3회 실패 시 "강제 소환" 처리, 방명록용 기록 문자열 생성.
- **렌더 래퍼**: 흔들림 변환 → 장르 render → 파티클 → 텍스트 → 플래시 순서. HUD(DOM) 갱신.

### 5.2 장르 모듈 인터페이스

```ts
interface GenreModule<St> {
  manifest: GenreManifest;
  init(ctx: GameContext): St;                       // 상태 생성
  update(st: St, dt: number, ctx: GameContext): void;
  render(st: St, g: CanvasRenderingContext2D, ctx: GameContext): void;
  down?(st: St, x: number, y: number, ctx: GameContext): void;
  move?(st: St, x: number, y: number, ctx: GameContext): void;
  up?(st: St, x: number, y: number, ctx: GameContext): void;
  cmd?(st: St, name: string, ctx: GameContext): void;
}
interface GameContext {
  config: InvitationConfig;                         // 이름·외형·사진
  fx: Fx; sfx: (id: SfxId) => void;
  hud(v: { hp?: string; label?: string }): void;
  controls(spec: ControlSpec[]): void;              // 하단 버튼/정보줄 선언 (장르가 DOM을 직접 만지지 않게)
  finish(r: { win: boolean; stars: 0|1|2|3 }): void;
  rand(): number;                                   // 시드 고정 가능 → 테스트·리플레이
}
```

- 데모와 달리 **전역 변수 금지**, 장르는 `St`만 바꾼다 → 장르 단위 테스트 가능.
- `rand()` 시드 주입으로 같은 입력이면 같은 결과 → 밸런스 자동 시뮬레이션 가능.

### 5.3 장르 매니페스트

```ts
interface GenreManifest {
  id: GenreId; version: number;
  title: string; genreLabel: string; tip: string;
  slots: { prologue: { count: 3; maxLen: 40 }; bossName?: { maxLen: 8 } };
  defaults: (c: InvitationConfig) => { title: string; lines: [who: string, say: string][]; clear: string };
  balance: Record<string, number>;                  // 기획 2가 조정하는 수치 (아래 표)
}
```

### 5.4 밸런스 수치 (데모 값 = 출발점)

| 장르 | 항목 | 데모 값 |
|---|---|---|
| RPG | 보스 HP / 크리티컬 / 보통 / 빗나감 | 40 / 12 / 7 / 3 |
| RPG | 크리티컬 구간 / 보통 구간 (바 기준) | ±7% / ±20% |
| RPG | 합동기 데미지 / 게이지 증가(크리·보통·빗나감) | 20 / 50·30·15 |
| RPG | 적 공격 예고 / 방어 입력 창 | 0.6~1.2초 / 0.5초 |
| 포격 | 포탄 / 클리어 조건 / 폭발 반경 | 5발 / 성벽 50% 이하 / 19px |
| 포격 | 중력 / 바람 범위 / 최대 속도 | 120 / -3~+3 / 150 |
| 레이싱 | 목표 거리 / 기본·부스트·충돌 속도 | 2400 / 170·240·70 |

목표: 첫 시도 클리어율 [70%], 평균 플레이 15초 ± 5초. 시드 고정 봇으로 1,000판 돌려 분포 확인.

### 5.5 성능 예산 (카톡 인앱, 중저가 안드로이드 기준)

| 항목 | 예산 |
|---|---|
| 하객 첫 화면(타이틀) 표시 | 1.5초 이내 (4G) |
| 첫 로드 JS (런타임+씬) | 60KB 이하 (gzip) |
| 장르 모듈 1개 | 30KB 이하 (gzip) |
| 게임 중 FPS | 45fps 이상 유지 |
| 웹폰트 | 픽셀 폰트는 사용 글자만 서브셋 |

---

## 6. 보안·개인정보

- 하객 페이지에는 **공개 스냅샷만** 노출. 소유자 ID, 원본 사진 경로, 결제 정보는 스냅샷에 넣지 않는다.
- 사진 EXIF 제거, 원본 버킷 비공개, 웹용 사진은 만료되는 서명 URL.
- 계좌번호는 베타에서 저장하지 않는다 (카카오페이 송금 링크만). 기존 저장소에 공개된 계좌번호·실명은 선행 정리에서 제거.
- 방명록: 레이트 리밋(IP+slug 기준 분당 3건), 금칙어 필터, 소유자 숨김, 신고.
- `noindex` 메타 + `robots.txt`로 청첩장 페이지 검색 노출 차단.
- 만료 후 30일 파기 정책은 약관·개인정보처리방침과 문구를 맞춘다 (사업 1과 협업).

---

## 7. 테스트·품질

- **단위**: zod 스키마, 장르 모듈(시드 고정 시뮬레이션: "봇이 70% 이상 클리어"를 CI 검사로).
- **E2E**: Playwright 모바일 뷰포트로 에디터 → 발행 → 하객 열람 → 정보 보기 → 방명록.
- **실기기**: 카톡 인앱(안드로이드 2종, iOS 2종), 삼성 인터넷, 사파리. W1-2 성능 스파이크 결과로 목록 확정.
- **모니터링**: Sentry(프론트·Edge), 게임 이벤트 로그(시작/클리어/실패/스킵/정보 보기) → 클리어율·스킵률 대시보드.

---

## 8. 담당과 일정 매핑

| 영역 | 담당 | W3-6 (11/16~12/13) | W7-8 (12/14~12/27) |
|---|---|---|---|
| 공통 런타임 + RPG | FE 1 | 런타임, RPG, 공통 씬 | 밸런스 반영, 성능 튜닝 |
| 포격 | FE 1 (+퍼블 지원) | — | 포격 장르 |
| 에디터·미리보기·공유 | FE 2 | 에디터 4단계, iframe 미리보기 | 카톡 공유, 발행 흐름 |
| 픽셀 UI·씬 레이아웃 | 퍼블 | 대화창·카드·HUD | 에셋 교체, 반응형 |
| 인증·데이터·사진 | BE 1 | 스키마, RLS, 업로드 파이프라인 | 방명록, 신고 |
| 결제·만료 | BE 2 | PG 연동(테스트 모드) | 웹훅 검증, 만료 Cron |
| 인프라·OG·관측 | BE 3 | 배포 파이프라인, OG 생성 | Sentry, 이벤트 로그 |

리스크: FE 1에 게임 전부가 몰린다. 포격이 밀리면 오픈 베타로 이동(기획서 일정과 동일). 퍼블이 장르 render 일부를 맡는 방안 검토.

---

## 9. 열린 질문 (11/12 리뷰에서 결정)

1. 배포: Vercel vs Cloudflare (비용, 국내 응답 속도 비교 필요)
2. PG: 포트원 경유 vs PG사 직접 (수수료, 심사 기간)
3. 에디터 UI: React 섬 vs Astro + 순수 TS (FE 2 숙련도 기준)
4. 무료 플랜 만료 기준: 발행일 기준 vs 예식일 기준
5. 하객 플레이 기록: 익명 집계만 할지, 방명록 닉네임과 연결할지
6. 레거시 청첩장 페이지(현재 GitHub Pages)를 언제 내릴지
