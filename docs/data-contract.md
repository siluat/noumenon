# Data Contract (Phase C)

#unreviewed

Noumenon의 두 축(Gleaner, Frontend) 사이에 흐르는 데이터 산출물의 **파일 구조와 소비 패턴**에 대한 계약 문서. **타입 정의의 진실은 Gleaner가 생성한 `.d.ts`** ([[gleaner-redesign]] § 6.4)이며, 이 문서는 그 산출물에 대한 **사람 언어의 설명서**다.

- 상위: [[noumenon]]
- 선행: [[feature-scope]]

---

## 0. 이 문서의 위상 (오해 방지)

### 0.1 진실의 출처는 raw-data, 코드 형태는 Gleaner 생성물

| 계층 | 위치 | 성격 |
|---|---|---|
| 1차 진실 | `ff14-raw-data/rawexd/*.csv` | 외부에서 오는 스키마의 단일 진실 (single source of truth) |
| 2차 (코드 형태) | `packages/gleaner/dist/types/noumenon.d.ts` | 1차 진실의 결정론적 코드 표현 |
| 사람 설명서 | **이 문서** | 2차의 의도/구조/소비 패턴을 사람 언어로 설명 |

**이 문서는 타입을 정의하지 않는다.** 본문의 TypeScript 블록은 "Gleaner가 생성하는 타입은 대략 이런 형태"라는 인용/예시이며, 실제 빌드되는 타입은 `.d.ts`에서 결정된다. 둘이 어긋나면 **이 문서를 갱신**해야지, `.d.ts`를 손으로 고쳐서는 안 된다.

### 0.2 이 문서가 다루는 것

- Gleaner 산출물의 **파일 구조** (어떤 JSON이 어디에 어떤 크기로 출력되는가)
- Summary / Detail **분리 정책**
- 각 엔티티가 다른 엔티티를 참조하는 **FK 표현 패턴** (embed × ID 하이브리드)
- Frontend의 **로드 전략** (초기/lazy)
- Frontend의 **FK 해석 구현 패턴**

타입 자체의 정의(필드 목록, 옵셔널 여부)는 [[feature-scope]]가 의도를 정하고, Gleaner의 `.d.ts`가 결정한다.

---

## 1. 파일 산출물 구조

Gleaner의 `extract` 명령이 만드는 디렉토리:

```
packages/gleaner/dist/
├── data/
│   ├── itemSummaries.json          ItemSummary[]                ~2MB
│   ├── item/
│   │   ├── 1.json                  Item                          ~2KB
│   │   ├── 2.json                  Item                          ~2KB
│   │   └── ...                     × 약 38,000개
│   ├── recipeSummaries.json        RecipeSummary[]              ~200KB
│   ├── recipe/
│   │   ├── 1.json                  Recipe                        ~500B
│   │   └── ...                     × 약 4,000개
│   ├── classJobs.json              ClassJob[]                    ~5KB (19개)
│   ├── baseParams.json             { id, name }[]                ~1KB
│   └── itemUICategories.json       ItemUICategory[]              ~10KB
├── types/
│   └── noumenon.d.ts               전 도메인 타입 (Gleaner 생성, 진실)
└── reports/
    ├── schema-diff.md              이전 빌드 대비 변경
    └── optional-analysis.md        데이터 기반 optional 판정 결과
```

### 배포 위치 권고

| 산출물 | 배포 | 근거 |
|---|---|---|
| `data/itemSummaries.json` | CDN (또는 정적 호스팅) | 2MB. 검색 페이지 진입 시 lazy load |
| `data/item/{id}.json` × 38k | CDN | 개별 lazy load |
| `data/recipeSummaries.json` | CDN | 200KB. Item 상세의 "제작법" 탭 진입 시 |
| `data/recipe/{id}.json` × 4k | CDN | 개별 lazy load |
| `data/classJobs.json` / `baseParams.json` / `itemUICategories.json` | Frontend `public/` 번들 | 합 ~16KB. 초기 로드 |
| `types/noumenon.d.ts` | Frontend가 import (workspace path 또는 publish) | 빌드 의존성 |

---

## 2. Summary / Detail 분리 정책

### 정책

- **Item, Recipe**: Summary와 Detail 분리. Summary는 목록/검색 화면용 축약. Detail은 한 엔티티당 한 파일.
- **ClassJob**: 분리 없음. 엔티티 수가 적으므로 단일 파일에 전체 직접 포함.

### Summary가 가져야 하는 것 (의도)

목록/검색 화면이 **상세 파일을 페치하지 않고도 한 행을 그릴 수 있을 만큼**의 필드. 예시(실제 정의는 `.d.ts` 참조):

```typescript
// 의도 인용 — 진실은 Gleaner의 .d.ts
export type ItemSummary = {
  id: number;
  name: string;
  icon: number;
  itemLevel: number;
  rarity: ItemRarity;
  itemUICategoryId: number;
  equipLevel: number;
};

export type RecipeSummary = {
  id: number;
  craftTypeId: number;
  itemResultId: number;
  recipeLevel: number;
  canHq: boolean;
  isSecondary: boolean;
};
```

Summary는 **객체 배열이 기본**. 실제 성능 이슈 시 Gleaner가 숫자 배열 압축 옵션을 제공할 수 있음 ([[gleaner-redesign]] § 10.3).

### ClassJob이 분리되지 않는 경계

엔티티 수 < 50 정도까지는 단일 파일 정책 유지. 그 이상으로 늘어나면 정책 재검토. 자동 판정 가능 (Gleaner가 엔티티 수를 보고 split).

---

## 3. FK 표현 패턴: embed × ID 하이브리드

### 3.1 두 갈래

**embed (값 해석을 Gleaner가 한 결과)**:
- 작은 참조 테이블의 핵심 정보를 도메인 객체에 직접 포함.
- Frontend가 추가 페치 없이 즉시 표시 가능.
- 예: Item의 `baseParam0..5`는 `{ name: "힘", value: 50 }` 객체. Item의 `itemUICategory`는 카테고리 meta 일부 embed.

**ID 참조**:
- 독립 엔티티 간 링크.
- Frontend가 상주 Map(`classJobMap`, `itemSummaryMap`)으로 해석.
- 예: Item의 `repairClassJob: number | null` (ClassJob.id 참조), Recipe의 `itemIngredient: number[]` (Item.id 배열).

### 3.2 어느 쪽을 쓰는가의 원칙

| 조건 | 선택 |
|---|---|
| 참조 대상이 작고(수십 개), 거의 모든 화면에서 같이 표시됨 | embed |
| 참조 대상이 크고(수천+), 별도 진입 가능한 엔티티 | ID 참조 |
| 자기참조 (Item ↔ Item) | ID 참조 (영구) |
| 보조 테이블 (BaseParam name, ItemUICategory meta) | embed |

### 3.3 예시

```
Item.baseParam0          → { name: "힘", value: 50 }     (embed)
Item.repairClassJob      → 5 (ClassJob.id)               (ID)
Item.repairItem          → 5594 (Item.id)                (ID, 자기참조)
Item.itemUICategory      → { id, name, icon, ... }       (embed)
Recipe.craftType         → 8 (ClassJob.id)               (ID)
Recipe.itemIngredient    → [5057, 5058, ...] (Item.id[]) (ID)
ClassJob.classJobParent  → 1 (ClassJob.id)               (ID, 자기참조)
```

---

## 4. 아이콘 표현

`icon` 필드는 **숫자 ID**. Frontend는 ID → URL로 변환하는 decoder를 둔다.

근거: raw가 icon path 문자열을 줄 때도 있고 ID만 줄 때도 있는데, 변환 알고리즘은 단순한 함수이므로 Frontend에서 처리하는 것이 깔끔. JSON에는 더 작은 표현(숫자)만 흐른다.

---

## 5. Frontend 소비 패턴

### 5.1 초기 부트스트랩

```typescript
// 앱 진입 시 (1회 로드)
const classJobs   = await fetch('/data/classJobs.json').then(r => r.json());
const baseParams  = await fetch('/data/baseParams.json').then(r => r.json());
const uiCategories = await fetch('/data/itemUICategories.json').then(r => r.json());

const classJobMap   = new Map(classJobs.map(cj => [cj.id, cj]));
const baseParamMap  = new Map(baseParams.map(bp => [bp.id, bp]));
const uiCategoryMap = new Map(uiCategories.map(c => [c.id, c]));
```

총 ~16KB. 이 Map들은 앱 수명 내내 메모리 상주.

### 5.2 검색 화면 진입 시

```typescript
const itemSummaries = await fetch('/data/itemSummaries.json').then(r => r.json());
const itemSummaryMap = new Map(itemSummaries.map(s => [s.id, s]));
// 약 2MB, 1회 로드 후 영구 캐시 (React Query staleTime: Infinity)
```

### 5.3 상세 진입 시

```typescript
// /item/:id 라우트
const item: Item = await fetch(`/data/item/${id}.json`).then(r => r.json());
```

### 5.4 FK 해석 (상세 페이지 렌더 시)

```typescript
// 예: Item 상세에서 "수리 가능 직업" 표시
const repairJob = item.repairClassJob
  ? classJobMap.get(item.repairClassJob) ?? null
  : null;
// 화면: repairJob?.name  // "목수"

// 예: repairItem 클릭 → 해당 아이템 페이지 이동
const repairItemSummary = item.repairItem
  ? itemSummaryMap.get(item.repairItem)
  : null;
```

추가 페치 없이 상세 페이지 대부분의 FK 표시 가능. 추가 왕복은 "제작 레시피 탭 열기" 정도.

### 5.5 검색 전략

클라이언트 메모리 선형 필터. 2MB summaries는 현 웹 환경에서 체감 문제 없음. 초성 검색 등 고급 기능은 [[frontend-plan]]에서 결정.

---

## 6. 엔티티별 메모 (Gleaner 생성 타입의 사람 설명)

`.d.ts`가 진실. 아래는 의도와 주의사항을 사람 언어로 적은 것.

### 6.1 Item

- raw `Item.csv`의 익명 아닌 전 필드 패스 스루 ([[feature-scope]] § 1).
- 약 58개 필드는 UI 노출 기준선이며, .d.ts 출력의 부분집합.
- `baseParam0..5` / `specialBaseParam0..5`는 슬롯 유지 ([[feature-scope]] § 5.1). 빈 슬롯은 optional로 표현됨.
- FK 컬럼: `repairClassJob`, `repairItem`, `glamourItem`, `useClassJob` — 모두 number | null.
- 알고리즘 변환 결과: `physicalDamage`, `magicalDamage` (사람 손맛 축약은 채택 안 함; UI 라벨은 Frontend 표시 라벨 맵에서).

### 6.2 Recipe

- raw `Recipe.csv` 패스 스루 + Gleaner enrich.
- `recipeLevel`, `maxQuality`는 Gleaner가 `RecipeLevelTable` flatten해 주입한 파생 필드.
- `itemIngredient`, `amountIngredient`는 raw의 8슬롯 고정 배열을 collapse한 가변 배열. 인덱스 동기화 보장.

### 6.3 ClassJob

- 19개 엔티티, 단일 파일 출력.
- `classJobParent`는 자기참조 FK (Gladiator → Paladin 같은 진화 관계).
- `nameEnglish`는 raw CSV의 타입 표기가 `byte`로 잘못된 케이스를 Gleaner가 `exd-header.json`으로 교정한 결과 ([[gleaner-redesign]] § 4.4).

### 6.4 보조 (BaseParam, ItemUICategory)

- `BaseParam`: id + name만. Item의 embed와 ClassJob.primaryStat 해석에 모두 활용.
- `ItemUICategory`: id, name, icon, majorOrder, minorOrder. Item에 embed될 수도 있고 별도 파일도 제공 (선택).

---

## 7. raw 변화 시 이 문서의 갱신 절차

1. raw에 변경 발생 → `gleaner extract` 실행.
2. `dist/reports/schema-diff.md`가 변화 표면화.
3. .d.ts와 데이터 JSON은 자동 갱신됨 (사람 손 안 탐).
4. **이 문서**는 사람 언어로 변화의 의도/배경을 추가 설명. 예: 새 필드의 도메인 의미, FK 패턴 분류 변경.
5. [[feature-scope]]의 MUST/NICE/LOW 분류 갱신 검토.
6. [[frontend-plan]]에서 새 필드의 UI 노출 여부 결정.

이 절차에서 어디에도 ".d.ts를 손으로 편집한다"는 단계가 없음에 주의.

---

## 8. 다음 단계

- [[gleaner-redesign]]: 이 산출물 구조를 만들어내는 Rust 코드 설계.
- [[frontend-plan]]: 이 산출물을 소비하는 Frontend 설계. 표시 라벨 맵 위치, 엔티티 그래프 탐색 UX.
