# Frontend Plan (Phase C)

#unreviewed

Noumenon Frontend(`apps/front`) 설계 문서. [[gleaner-redesign]]이 생성하는 JSON과 `.d.ts`(=타입의 진실)를 소비해 "아이템 중심의 엔티티 그래프 탐색" UX를 제공하는 웹 서비스를 설계한다. [[data-contract]]는 그 산출물에 대한 사람 설명서로 참조.

- 상위: [[noumenon]]
- 선행: [[feature-scope]], [[data-contract]], [[gleaner-redesign]]

---

## 1. 설계 원칙

### 1.1 "아이템 중심의 탐색 그래프"가 Frontend의 정체성
"아이템 전용 검색/상세 뷰어"를 골격으로 두고, 그 위에 **엔티티 FK를 클릭 가능한 링크**로 만들어 탐색 동선을 넓힌다. 핵심 동선:

```
검색 → Item 상세 → (repairItem) → 다른 Item 상세
                ↘ (제작 레시피 탭) → Recipe 상세 → (ingredient) → 다른 Item 상세
                ↘ (repairClassJob) → ClassJob 상세 → (starting weapon) → 다른 Item 상세
```

URL이 곧 경로. "뒤로가기" 한 번으로 돌아가기가 가능해야 한다.

### 1.2 검증된 레이아웃 + 현대 UX
검증된 레이아웃/분류(SearchInput + 가상화 리스트 + ItemDetail)는 **기본 골격으로 유지**. 다만 다음은 재평가:
- 기술 스택 (React 19 + Vite + React Query — 현대 React 표준)
- UX 세부 (초성 검색, 다크 모드, 키보드 내비, URL 딥링크, 모바일 가독성)

### 1.3 "추가 페치 없이 표시" 원칙
초기 로드된 summaries + ClassJob + BaseParam 테이블로 **상세 페이지의 대부분 FK**를 즉시 해석해 렌더. 상세 JSON 페치는 1회(엔티티 본체)로 제한. 추가 왕복이 필요한 것은 "제작 레시피 탭 열기" 정도.

### 1.4 검색 엔진 없음
클라이언트 메모리 선형 필터. 2MB summaries는 현 웹 환경에서 체감 문제 없음. 고급 검색 기능(초성/오타/다국어)은 UX 레이어에서 결정.

### 1.5 타입은 import만, 표시 라벨은 Frontend 책임

raw-data가 single source of truth ([[feature-scope]] 원칙 1)이므로 Frontend는 타입을 **수동으로 정의하지 않는다**. Gleaner가 생성한 `.d.ts`를 그대로 import한다.

대신, 사람 손맛 축약 라벨(`physDamage`, `modifierHp` 등)이나 한국어 표시명 같은 "사람이 읽는 표현"은 Frontend의 **표시 라벨 맵**에서 관리한다. 데이터 스키마(필드명)와 화면 라벨은 분리.

```typescript
// src/labels/item.ts (예시)
export const ITEM_FIELD_LABELS: Partial<Record<keyof Item, string>> = {
  physicalDamage: '물리 공격력',     // raw 알고리즘 결과는 physicalDamage
  magicalDamage: '마법 공격력',      // 데이터 필드명은 magicalDamage, 화면 표기는 자유
  blockRate: '방어 발동',
  // ...
};
```

이 분리 덕에 raw가 새 필드를 추가해도 기본 표시는 자동으로 가능(필드명 그대로 노출)하고, 한국어 라벨이 필요해진 시점에 이 맵에 한 줄만 추가하면 된다.

---

## 2. 기술 스택 권고안

### 2.1 권고 스택

| 영역 | 선택 | 근거 | 대안 |
|---|---|---|---|
| **언어** | TypeScript 5.x | Gleaner가 `.d.ts` 자동 생성. 당연 | - |
| **UI 프레임워크** | **React 19** | 생태계 가장 풍부. 검증된 컴포넌트 골격을 가장 쉽게 가져옴 | Solid/Svelte (학습 비용, 자산 호환성) |
| **번들러/개발 서버** | **Vite 5+** | CRA 공식 디프리케이트. HMR/빌드 속도 | Next.js(SSR 불필요), Turbopack(덜 성숙) |
| **라우터** | **React Router v7** (data mode) | 엔티티 그래프 탐색의 기반 = URL. loader 패턴이 "상세 진입 시 JSON 페치"와 딱 맞음 | TanStack Router(타입 안전 우수, 학습 비용) |
| **서버 상태** | **TanStack Query (React Query) v5** | JSON 페치/캐시만 하면 됨. React Router loader와 조합해 "라우트 전환 = 캐시 키" | SWR (기능 좁음), 직접 `fetch` + useState(캐싱 재발명) |
| **클라이언트 상태** | **Zustand 또는 useState/useReducer** | 필터/검색어 정도만 전역. Redux는 과잉 | Redux Toolkit(현 규모에 과잉) |
| **스타일** | **CSS Modules + CSS Variables** 또는 **Tailwind CSS v4** | 빌드 단순, 다크모드 용이 | Emotion(런타임 비용, React 19와의 궁합 재검증 필요), Styled Components(동일) |
| **가상화 리스트** | **@tanstack/react-virtual** | `react-virtualized`의 현대적 후계자. 훅 기반, 타입 좋음 | react-window(소유자 동일 계열, 기능 차이 거의 없음) |
| **아이콘 폰트/직업 아이콘** | **자체 decoder 모듈** | FF14 icon ID → S3 URL 변환은 검증된 단순 알고리즘 | - |
| **폼 상태** | **없음** (검색창 하나) | controlled input 하나로 충분 | - |
| **테스트** | **Vitest + @testing-library/react** | Vite 네이티브, Jest 호환 | Jest(Vite 환경에서 조합 번거로움) |
| **린트/포맷** | **Biome** 또는 **ESLint + Prettier** | Biome은 단일 바이너리로 경량. ESLint는 관습 | - |
| **배포** | Netlify 또는 Cloudflare Pages | 정적 사이트 + 커스텀 도메인. CI 연동 간단 | Vercel(Next.js에 최적), S3 정적 호스팅(직접 관리) |

### 2.2 **최소 의존성 원칙**
권고 스택은 "개인 프로젝트에 과하지 않은 최소 셋"을 목표로 함. React Query + Zustand + React Router 3개가 사실상 핵심. 나머지는 교체 가능.

---

## 3. 라우트 & 페이지 구조

### 3.1 라우트 맵

| 경로 | 페이지 | loader (React Router) | 용도 |
|---|---|---|---|
| `/` | `HomePage` | summaries 프리페치 | 랜딩/소개 + 빠른 검색 진입 |
| `/search` | `SearchPage` | summaries 보장 | 필터/정렬 가능한 검색 |
| `/search?q=...` | 동일 | 동일 | URL 쿼리 = 검색어 (딥링크) |
| `/item/:id` | `ItemDetailPage` | `item/{id}.json` | 아이템 상세 |
| `/recipe/:id` | `RecipeDetailPage` | `recipe/{id}.json` | 레시피 상세 |
| `/classjob/:id` | `ClassJobDetailPage` | classJobs.json 읽음 | 직업 상세 |
| `/classjob` | `ClassJobListPage` | classJobs.json 읽음 | 19개 직업 일람 |
| `*` | `NotFoundPage` | - | 404 |

### 3.2 초기 부트스트랩 (`loader` or `root loader`)
앱 진입 시(모든 라우트 공통) 다음을 프리페치:
- `classJobs.json` (5KB)
- `baseParams.json` (1KB)
- `itemUICategories.json` (10KB, embed 선택에 따라 생략 가능)
- `itemSummaries.json` (2MB, lazy 진입 시 prefetch 가능)

React Router v7의 `root loader`에서 Promise를 `defer`로 묶어 네트워크 지연에 강한 UX 제공.

### 3.3 딥링크 원칙
**모든 상태는 URL에 있거나, 없어도 된다.**
- 검색어: `?q=...`
- 카테고리 필터: `?category=...`
- 정렬: `?sort=...`
- 페이지네이션: 가상화 리스트이므로 페이지 없음
- 다크 모드: localStorage (URL 불필요)
- 열린 탭 (Item 상세의 "제작 레시피" 탭 등): `?tab=...` (선택)

뒤로가기/새로고침이 같은 화면을 재현해야 한다.

---

## 4. 데이터 로딩 전략

### 4.1 3단계 로드

```
1. 앱 진입 시 (root loader)
   - classJobs.json         ← 상주 (Map으로)
   - baseParams.json        ← 상주 (Map으로)
   - itemUICategories.json  ← 상주 (선택)

2. 검색/리스트 진입 시 (route loader)
   - itemSummaries.json     ← 처음 필요할 때. 이후 React Query 캐시

3. 상세 진입 시 (route loader)
   - item/{id}.json / recipe/{id}.json / classJob 단건
   - React Query staleTime=∞ (JSON은 빌드 결과물이므로 런타임 변화 없음)
```

### 4.2 캐시 정책
- JSON은 빌드 결과물 → **영구 캐시**. `staleTime: Infinity`.
- 버전 업그레이드 시 CDN 경로에 버전 segment 포함 (`/data/v1/...`). Gleaner가 `--version` 태그 지원.
- Frontend는 CDN URL base를 환경변수로. 로컬/스테이징/프로덕션 분리.

### 4.3 오프라인/PWA
범위 밖. 나중에 필요하면 service worker로 summaries를 장기 캐시하는 정도.

---

## 5. 컴포넌트 인벤토리

### 5.1 검증된 골격 (내부 구현은 현대화)

| 컴포넌트 | 역할 | 재작성 포인트 |
|---|---|---|
| `SearchInput` | 검색창 | 디바운스 + IME(한글 조합) 처리 |
| `SearchResult` | 가상화 리스트 | `react-virtualized` → `@tanstack/react-virtual` |
| `ItemInline` | 리스트 행 | 동일 |
| `ItemDetail` | 상세 레이아웃 | 탭 구조 도입 (개요/장착/제작/환상) |
| `ItemIcon` | 아이콘 | decoder 계승, `<img loading="lazy">` |
| `ItemName` | 레어도별 색상 표시 | `ItemRarity` enum 기반 CSS var |
| `BonusParameters` | baseParam0~5 / specialBaseParam0~5 | 동일 |
| `MateriaSlots` | 메테리아 슬롯 | 동일 |
| `CraftingAndRepair` | 제작/수리 정보 | **링크 업그레이드**: repairItem, repairClassJob 클릭 시 해당 엔티티로 이동 |
| `StorageInformation` | 저장/거래 플래그 | 동일 |
| `JobIconMorph` | 직업 아이콘 변형 | CSS transform 기반 단순화 |

### 5.2 신규

| 컴포넌트 | 용도 |
|---|---|
| `RecipeInline` | 레시피 한 줄 요약 (Item 상세의 "제작법 탭" 안에서) |
| `RecipeDetail` | Recipe 전체 뷰 (재료 리스트 + 난이도 파라미터) |
| `IngredientList` | `itemIngredientIds × amountIngredients` 배열을 ItemInline 행들로 |
| `ClassJobBadge` | 직업 아이콘 + 약어 + 이름 (FK 해석된 자리) |
| `ClassJobDetail` | ClassJob 전체 뷰 (Role/PrimaryStat/Starting weapon/Modifier 등) |
| `EntityGraphBreadcrumb` | 탐색 경로 표시 (선택, UX 실험) |
| `ThemeToggle` | 다크/라이트 전환 |
| `KeyboardHelp` | `?` 키로 단축키 안내 (선택) |

---

## 6. UX 업그레이드 포인트

업그레이드 여지를 MUST/NICE/OUT으로 분류.

### 6.1 MUST (초기 릴리스에 포함)

| 항목 | 설명 |
|---|---|
| **URL 딥링크** | 검색어/카테고리/정렬을 전부 URL 쿼리로 |
| **FK 링크 클릭** | 상세 페이지의 FK 값(수리직업, 재료 등)을 전부 클릭 가능하게 |
| **키보드 내비** | `/`로 검색창 포커스, `esc`로 검색창 초기화, `↑↓`로 결과 이동, `enter`로 상세 진입 |
| **다크 모드** | CSS variables + `prefers-color-scheme` + 토글 |
| **모바일 레이아웃** | 데스크톱/모바일 모두 지원하는 반응형 레이아웃 |
| **아이콘 lazy loading** | 리스트 38k 행 전부 요청하지 않기 |

### 6.2 NICE (M2 이후)

| 항목 | 설명 |
|---|---|
| **초성 검색** | "검"으로 "검술사" 매칭. 한국어 사용성 향상 |
| **히스토리 breadcrumb** | 탐색 경로를 UI로 노출 (브라우저 히스토리 외에 in-app 표시) |
| **카테고리 트리 네비** | `ItemUICategory`의 major/minor order 활용한 사이드바 |
| **Recipe 역탐색** | "이 재료가 쓰이는 레시피" 리스트 (인덱스를 클라이언트에서 한 번 구축) |
| **비교 뷰** | 2~3개 Item을 나란히 띄우기 |

### 6.3 OUT (명확히 제외)

| 항목 | 사유 |
|---|---|
| 다국어 | 데이터가 한국어 단일 (feature-scope § 1 근거) |
| 로그인/개인화 | 서비스 성격상 불필요 |
| SSR | 정적 JSON + SPA로 충분. SEO는 소수 페이지만 prerender로 보완 가능 |
| 고급 검색 엔진 (Algolia 등) | 2MB 선형 필터로 충분 |
| 네이티브 앱 | 범위 밖 |

---

## 7. 상태 관리 상세

### 7.1 3분류
- **서버 데이터**: React Query. 캐시 키는 JSON 파일 경로.
- **URL 상태**: React Router의 `useSearchParams`, `useParams`.
- **로컬 UI 상태**: `useState`/`useReducer` 또는 Zustand (테마/모달/토스트 정도).

### 7.2 상주 Map 구현 패턴

```typescript
// lib/lookups.ts
import { QueryClient } from '@tanstack/react-query';

export const getClassJobMap = async (qc: QueryClient) => {
  const list = await qc.fetchQuery({
    queryKey: ['classJobs'],
    queryFn: () => fetch('/data/classJobs.json').then(r => r.json()),
    staleTime: Infinity,
  });
  return new Map<number, ClassJob>(list.map(cj => [cj.id, cj]));
};
```

[[data-contract]] § FK 해석 패턴에서 제시한 방식을 React Query 위에 얇게 래핑.

### 7.3 Redux를 안 쓰는 근거
- 동기 액션만 있고 비동기 흐름이 간단 (JSON fetch가 전부) → Saga 불필요.
- 정규화된 스토어가 필요할 만큼의 엔티티 그래프 동시성 없음.
- DevTools의 타임트래블 디버깅보다 React Query DevTools가 이 프로젝트에 더 직접적으로 유용.

---

## 8. 검색 UX

### 8.1 기본 검색 (MUST)
- `SearchInput`은 controlled, 300ms 디바운스.
- 입력 → `useSearchParams`로 `?q=` 업데이트 → `useMemo`로 summaries 필터.
- 한글 IME 조합 중에는 필터를 일시 중단 (`compositionstart/end` 이벤트).

### 8.2 초성 검색 (NICE)
- 로드된 summaries를 클라이언트에서 1회 전처리: `name` → `chosung(name)` 캐시.
- 한국어 전용 함수 구현 (외부 의존 없이 수십 줄).

### 8.3 카테고리 필터 (MUST)
- `ItemUICategory`별 그룹. 상주 Map에서 가져와 드롭다운.
- `?category=<id>`로 URL에 반영.

### 8.4 정렬 (MUST)
- 이름/아이템 레벨/레어도 3종. `?sort=itemLevel,desc` 등.

### 8.5 결과 수 기대치
38,000 아이템 선형 필터. Chrome 기본 `.filter + includes`로 5~15ms. 문제없음.

---

## 9. 시각 디자인 방향

### 9.1 테마 토큰
```css
:root {
  --color-bg: #ffffff;
  --color-text: #111111;
  --color-accent: #4c7eff;
  --color-rarity-0: #888;  /* 회색 */
  --color-rarity-1: #fff;  /* 흰색 */
  --color-rarity-2: #6bf;
  --color-rarity-3: #9bf;
  --color-rarity-4: #ffc;
  --color-rarity-5: #fca;
  --color-rarity-7: #f8f;
}
[data-theme="dark"] {
  --color-bg: #121214;
  --color-text: #eaeaea;
  /* ... */
}
```

레어도 색상 매핑을 CSS variables로 일반화.

### 9.2 타이포그래피
- 본문: 시스템 폰트 스택 (`-apple-system`, Apple SD Gothic Neo).
- 모노스페이스: 숫자 정렬 필요한 파라미터 테이블에만.

### 9.3 레이아웃 원칙
- 최대 `1200px` 중앙 정렬.
- 상세 페이지는 2-column (좌: 아이콘/이름/기본, 우: 파라미터/탭).
- 모바일(<768px)은 1-column, 탭은 아코디언화.

---

## 10. 성능 & 접근성 목표

### 10.1 성능 예산
- 초기 JS 번들 < 200KB gzipped (React + React Router + React Query + Zustand + 앱 코드).
- 초기 렌더까지 < 1s (3G Slow 기준 제외, 광대역 기준).
- `itemSummaries.json` (2MB)은 필요한 라우트에서 lazy. 검색 페이지 진입 후 `< 500ms` 내 인터랙티브.
- 리스트 스크롤 60fps 유지 (가상화).

### 10.2 접근성
- 모든 아이콘 `alt` 제공 (ItemName 텍스트 재사용).
- 레어도 색상은 텍스트 + 색으로 이중 표현.
- 포커스 가시성 유지, 키보드만으로 탐색 가능.
- 대비비 WCAG AA 준수 (다크/라이트 양측 모두).

---

## 11. 디렉토리 구조 (제안)

### 11.1 타입 import 전략 (원칙 의존)

수동 작성 `src/types/` 디렉토리는 **두지 않는다** (=[[feature-scope]] 원칙 1, [[gleaner-redesign]] 원칙 2의 직접 귀결). Gleaner가 생성한 `.d.ts`를 workspace path로 직접 import.

```jsonc
// apps/front/tsconfig.json
{
  "compilerOptions": {
    "paths": {
      "@noumenon/types": ["../../packages/gleaner/dist/types/noumenon"]
    }
  }
}
```

```typescript
// 사용 측
import type { Item, Recipe, ClassJob, ItemSummary } from '@noumenon/types';
```

`.d.ts`는 Gleaner의 빌드 산출물이므로 git에 체크인할지 여부는 별도 결정 (CI에서 매번 빌드한다면 미체크인, 의존하는 PR 검토 편의를 위해선 체크인). 어느 쪽이든 **Frontend 측에서 손으로 편집하지 않는다**가 절대 규칙.

### 11.2 디렉토리

```
apps/front/
├── public/
│   └── data/
│       ├── classJobs.json
│       ├── baseParams.json
│       └── itemUICategories.json   # 소형 파일만 번들 (2MB 넘는 것은 CDN)
├── src/
│   ├── main.tsx
│   ├── App.tsx
│   ├── router.tsx
│   ├── (types/ 디렉토리 없음 — @noumenon/types로 import)
│   ├── labels/
│   │   ├── item.ts                 # 한국어/축약 표시 라벨 맵 (§ 1.5)
│   │   ├── recipe.ts
│   │   └── classJob.ts
│   ├── lib/
│   │   ├── lookups.ts              # Map 빌드 헬퍼
│   │   ├── decoder.ts              # icon id → URL
│   │   ├── chosung.ts              # 초성 검색 (NICE)
│   │   └── fetch.ts                # 공용 JSON fetcher
│   ├── pages/
│   │   ├── HomePage.tsx
│   │   ├── SearchPage.tsx
│   │   ├── ItemDetailPage.tsx
│   │   ├── RecipeDetailPage.tsx
│   │   ├── ClassJobListPage.tsx
│   │   ├── ClassJobDetailPage.tsx
│   │   └── NotFoundPage.tsx
│   ├── components/
│   │   ├── search/
│   │   │   ├── SearchInput.tsx
│   │   │   ├── SearchResult.tsx
│   │   │   └── ItemInline.tsx
│   │   ├── item/
│   │   │   ├── ItemIcon.tsx
│   │   │   ├── ItemName.tsx
│   │   │   ├── ItemDetail.tsx
│   │   │   ├── BonusParameters.tsx
│   │   │   ├── MateriaSlots.tsx
│   │   │   ├── CraftingAndRepair.tsx
│   │   │   └── StorageInformation.tsx
│   │   ├── recipe/
│   │   │   ├── RecipeInline.tsx
│   │   │   ├── RecipeDetail.tsx
│   │   │   └── IngredientList.tsx
│   │   ├── classjob/
│   │   │   ├── ClassJobBadge.tsx
│   │   │   └── ClassJobDetail.tsx
│   │   └── ui/
│   │       ├── ThemeToggle.tsx
│   │       └── KeyboardHelp.tsx
│   ├── styles/
│   │   ├── tokens.css              # CSS variables
│   │   └── reset.css
│   └── env.ts                      # CDN base URL 등
├── tests/
│   └── (컴포넌트 단위 테스트)
├── index.html
├── vite.config.ts
├── tsconfig.json
└── package.json
```

---

## 12. 오픈 퀘스천 (Phase D 결정)

1. **스타일링**: Tailwind CSS v4 vs CSS Modules — 팀(=개인) 선호로 결정. 기본 권고: **CSS Modules + variables** (의존성 최소, 빠른 빌드).
2. **CDN 결정**: 기존 S3 버킷 재사용 vs `noumenon`으로 신규 생성. AWS 비용/DNS 관점 결정 필요.
3. **배포 타깃**: Netlify vs Cloudflare Pages. 개인 프로젝트 부담 고려.
4. **아이콘 빌드 포함 여부**: 아이콘 88,551개(2.8GB)는 CDN 유지 확정(feature-scope § 4 근거). 다만 직업 아이콘 19개만 번들에 포함해 초기 UX 개선 가능.
5. **Recipe 역탐색 인덱스**: "이 재료가 쓰이는 레시피" 인덱스를 Gleaner가 미리 만들지(서버 비용) vs Frontend가 런타임에 1회 구축(초기 지연)지. 현재 제안: **Frontend 런타임 구축** (빌드 파이프라인 단순).

---

## 13. 다음 단계

- Phase D `roadmap.md`: 이 계획의 MUST 항목들과 [[gleaner-redesign]]의 M1~M8을 하나의 실행 순서로 엮는다. 의존성: **Gleaner M6(전체 파이프라인) → Frontend 실작업 가능**. 그 전까진 Gleaner 출력 샘플 1~2개만으로 Frontend 프로토타이핑.
