# Roadmap (Phase D)

Noumenon 재시작의 **시간순 실행 계획**. Phase C 문서들이 *무엇을 만드는가*를 정의했다면, 이 문서는 *어떤 순서로 만들고 각 단계의 완료 기준이 무엇인가*를 정의한다.

- 상위: [[noumenon]]
- 선행: [[current-state-audit]], [[feature-scope]], [[data-contract]], [[gleaner-redesign]], [[frontend-plan]]
- 대상 저장소: **`github.com/siluat/noumenon`** (신규, public)

---

## 0. 이 문서의 위상

| 종류 | 시간성 | 갱신 빈도 | 예시 |
|---|---|---|---|
| **Spec** ([[gleaner-redesign]], [[frontend-plan]] 등 Phase C) | 시간 무관 | 거의 변하지 않음 | "Layer 1은 순수함수다", "타입은 import만" |
| **Roadmap** (이 문서) | 시간순 | 단계 완료마다 | "D-2 완료 = `gleaner schema`로 단일 CSV 처리됨" |

> 💡 **인사이트 — 왜 Spec과 Roadmap을 섞지 않는가**
> 
> 두 문서를 한 곳에 두면 spec이 자꾸 *"지금 어디까지 했지?"* 같은 진행 상태 노이즈에 오염됩니다. 6개월 뒤 문서를 다시 읽을 때 spec은 여전히 유효해야 하지만, roadmap은 그 시점에 이미 대부분 끝나 있어야 합니다. 이 분리는 RFC(spec) ↔ project board(execution) 패턴과 같은 발상이며, 한 번 익숙해지면 이후 모든 프로젝트에서 같은 분리를 자연스럽게 적용하게 됩니다.

---

## 1. Phase D 전제 (사용자 결정 사항)

| # | 결정 | Roadmap에 미친 영향 |
|---|---|---|
| 1 | 모노레포에서 **분리** | D-1에 신규 레포 부트스트랩 + web-projects 정리 포함 |
| 2 | **완전히 새로 시작** (history 미보존) | 기존 코드는 *암묵 지식 추출 reference*로만 사용 (D-0). 코드 복사 없음 |
| 3 | librarian **폐기**, 필요 시 재생성 | D-1에서 `noumenon-librarian` 디렉토리 삭제. roadmap 산출물에 미포함 |
| 4 | 신규 레포 이름 = **`siluat/noumenon`**, public | D-1 부트스트랩의 첫 행동 |

> 💡 **인사이트 — "완전히 새로 시작"이 roadmap에 미치는 효과**
> 
> 기존 Rust 코드를 `subtree split`했다면 roadmap의 D-2(Layer 1)는 *"기존 builder.rs 분할 리팩토링"*이 됐을 것입니다. 새로 시작하므로 D-2는 *"Phase C spec 그대로 신규 구현"*이 됩니다. 이게 더 빠릅니다 — 기존 코드의 의도와 spec의 의도가 충돌하는 지점에서 *"무엇을 살릴지"* 판단할 필요가 없기 때문입니다. 단, 기존 코드만 알고 있던 *암묵 지식*(예: 특정 CSV의 인코딩 코너케이스, exd-header 파싱 트릭)을 잃을 위험이 있어, D-0 단계에서 한 번 훑고 노트로 보존합니다.

---

## 2. 마일스톤 의존 그래프

```
D-0 (Pre-flight)
  │
  ▼
D-1 (신규 레포 부트스트랩 + web-projects 정리)
  │
  ├──────────────────────────────────────┐
  ▼                                      ▼
D-2 (Gleaner Layer 1)              (Frontend 작업은 D-5 이후 본격화,
  │                                  그 전엔 D-1 골격까지만)
  ▼
D-3 (Gleaner Layer 2)
  │
  ▼
D-4 (Gleaner Layer 3 + extract 파이프라인)
  │
  ▼
D-5 (TypeScript 출력 + Schema Diff)  ◄─── Frontend 진입점
  │
  ▼
D-6 (Frontend 골격)
  │
  ▼
D-7 (Search + Item 상세)
  │
  ▼
D-8 (Recipe/ClassJob 상세 + UX MUST)
  │
  ▼
D-9 (배포 + 외부 공개)
  │
  ▼
D-10 (NICE 기능, 보류)
```

**병렬화 가능 구간**: D-1까지 끝내면 D-2(Gleaner)와 D-6 골격(Frontend)을 *부분적으로* 병렬 진행할 수 있다. 단, Frontend가 실제 데이터를 소비하려면 D-5(타입 출력)가 필요하므로, **사실상 단일 경로로 진행하는 게 안전**하다.

> 💡 **인사이트 — 왜 단일 경로 권장인가**
> 
> 1인 프로젝트에서 병렬은 거의 항상 함정입니다. *"Frontend도 좀, Gleaner도 좀"* 하다 보면 두 트랙 모두 미완으로 남고 컨텍스트 스위칭 비용만 늘어납니다. **Gleaner 출력이 안정될 때까지 Frontend를 손대지 않는다**는 규율이 길게 보면 더 빠릅니다.

---

## 3. D-0 — Pre-flight: 암묵 지식 추출

**목표**: 기존 코드/환경에만 있던 암묵 지식을 신규 레포에 가져갈 수 있는 형태로 보존.

### 산출물

- [ ] **`docs/known-quirks.md`** (dotori → 신규 레포로 이전 예정)
  - 기존 `web-projects/packages/noumenon-gleaner/src/schema/builder.rs`(710줄)에서 발견되는 *코너케이스 처리* 메모
  - CSV 파싱 시 마주쳤던 비정상 행/타입 표기 사례
  - `exd-header.json`에서 알게 된 포맷 (필드 위치, 타입 표기 충돌 사례)
- [ ] **`raw-data` 환경 재확인 메모**
  - `/Users/siluat/Projects/ff14-raw-data` 실제 위치, CSV 인코딩, 줄 끝 문자
  - `exd-header.json` 위치, 크기 (2.6MB), 7,777 테이블 가정의 실측 확인
- [ ] **(선택) gubal-front의 `Item` 58필드 목록 캡처**
  - [[feature-scope]] § 1의 MUST/NICE 분류와 실제 필드명 alignment 확인용
  - 기존 gubal 레포의 `src/types/Item.ts` 1회 스냅샷

### 완료 기준

- 신규 레포에서 *기존 코드를 다시 열어보지 않고도* D-2 구현을 시작할 수 있을 만큼의 정보가 dotori에 남았다.

### 예상 작업량

반나절 ~ 1일. 기존 코드 *전체를 읽지 않고* 코너케이스 주석/이슈/특이 분기만 grep으로 훑는다.

> 💡 **인사이트 — Pre-flight를 분리한 이유**
> 
> "어차피 spec대로 짜면 되니까 기존 코드는 잊자"는 유혹이 있지만, **710줄 `builder.rs`의 어딘가에는 *분명히* spec에 안 적힌 raw 데이터의 기벽 처리가 있습니다.** 이걸 잊으면 D-2/D-3 구현 중에 같은 함정에 다시 빠지고, 두 번째에 더 잘 풀리리란 보장도 없습니다. Pre-flight는 *"실패한 시도라도 거기서 배운 것은 가져간다"*는 원칙의 실천입니다.

---

## 4. D-1 — 신규 레포 부트스트랩 + web-projects 정리

**목표**: `siluat/noumenon` 레포가 살아 있고, `web-projects`에서 Noumenon 흔적이 깨끗이 정리됐다.

### 4.1 신규 레포 생성

- [ ] GitHub: `siluat/noumenon` (public) 생성
- [ ] 로컬 clone, 첫 commit은 `chore: init bun + turbo workspace`
- [ ] 기본 디렉토리:
  ```
  noumenon/
  ├── apps/
  │   └── front/                  # Vite 골격은 D-6에서
  ├── packages/
  │   └── gleaner/                # Cargo init은 D-2에서
  ├── docs/
  │   ├── feature-scope.md        # Phase C 이관본
  │   ├── data-contract.md
  │   ├── gleaner-redesign.md
  │   ├── frontend-plan.md
  │   ├── roadmap.md              # 이 문서 이관본
  │   └── known-quirks.md         # D-0 산출물
  ├── package.json                # workspaces: ["apps/*", "packages/*"]
  ├── turbo.json
  ├── biome.json
  ├── lefthook.yml
  ├── tsconfig.base.json
  ├── .gitignore
  ├── .github/workflows/ci.yml
  └── README.md
  ```

### 4.2 인프라 설정

- [ ] **bun**: `bun init`으로 root `package.json` + workspaces
- [ ] **turbo**: `web-projects/turbo.json` 참고해 작성. tasks: `build`, `test`, `dev`, `check-types`, `format-and-lint`
- [ ] **Biome**: `web-projects/biome.json` 복사 후 프로젝트 이름만 조정
- [ ] **lefthook**: 동
- [ ] **tsconfig.base.json**: `web-projects/packages/typescript-config/`의 base 설정을 inline. 이 패키지를 따로 옮기지 않음 (사용처가 noumenon 한 곳뿐이므로 inline이 더 간단)
- [ ] **GitHub Actions CI**: `bun install` → `turbo run check-types format-and-lint test` 1단계 워크플로
- [ ] **Cargo**: `packages/gleaner/Cargo.toml` 골격만 (실제 의존성은 D-2)

### 4.3 README.md (외부 첫 인상)

- [ ] 프로젝트 한 줄 정의 ("FF14 데이터 탐색 서비스")
- [ ] 기술 스택 (bun + turbo + Vite + Rust)
- [ ] 디렉토리 구조 가이드
- [ ] 로컬 개발 시작 명령
- [ ] *"Spec은 `docs/`에 있다"* 라인 (협력자 안내)

### 4.4 Phase C 문서 이관

- [ ] dotori `01_Projects/noumenon/`의 5개 문서를 신규 레포 `docs/`로 복사
- [ ] dotori 측 마스터 사본은 유지 (작성 이력 보존). 향후 갱신은 `docs/`만.
- [ ] dotori `noumenon.md` 프로젝트 허브 로그 갱신

### 4.5 web-projects 측 정리

- [ ] `apps/noumenon-front/` 디렉토리 제거
- [ ] `packages/noumenon-gleaner/` 디렉토리 제거
- [ ] `packages/noumenon-librarian/` 디렉토리 제거 (사용자 결정 #3)
- [ ] root `package.json` 자동 갱신 확인 (workspaces glob이 자동 축소)
- [ ] `bun install` 재실행, lockfile 갱신
- [ ] commit message: `chore: extract noumenon to dedicated repo`

### 완료 기준

- 신규 레포에서 `bun install && turbo run check-types`가 0 에러로 통과한다.
- web-projects에서 `apps/noumenon-front`, `packages/noumenon-*` 셋이 사라졌고 다른 패키지의 빌드가 깨지지 않는다.
- 신규 레포 README의 "로컬 개발 시작" 절차가 실행 가능하다.

### 예상 작업량

1일.

> 💡 **인사이트 — `tsconfig-config` 패키지를 inline으로 옮기는 결정**
> 
> 모노레포의 `@siluat/typescript-config` 패키지는 *여러 프로젝트가 공통 base를 공유*하기 위한 것입니다. Noumenon이 분리된 후에는 이 base를 쓰는 프로젝트가 한 곳뿐이므로, **패키지로 추상화할 필요가 사라집니다**. 같은 정보를 `tsconfig.base.json` 한 파일로 inline하는 게 더 단순하고, 의존성이 한 줄 줄어듭니다. *"두 군데 이상에서 쓰일 때만 추상화한다"*는 일반 원칙(rule of three의 변형)을 적용한 결정입니다.

---

## 5. Gleaner 단계 (D-2 ~ D-5)

[[gleaner-redesign]] § 9의 M1~M8 마일스톤을 이 4단계로 그룹화. **신규 구현이므로 M1의 "기존 builder.rs 분할" 부분은 D-0의 암묵 지식 추출로 대체됨**.

### 5.1 D-2 — Gleaner Layer 1: Pure Parser

**목표**: 단일 CSV 1개를 파싱해 `CsvData { schema, records }`를 만들 수 있다.

**대응 마일스톤**: M1 (재해석) + M2

#### 산출물

- [ ] `packages/gleaner/src/`의 다음 모듈 신규 작성:
  - `parse/csv_header.rs` — 3행 헤더 파싱
  - `parse/type_inference.rs` — `FieldType` 결정
  - `parse/normalize.rs` — 결정론적 필드명 알고리즘 (오버라이드 테이블 없음)
  - `parse/row.rs` — `RawValue` 변환
  - `parse/exd_header.rs` — `exd-header.json` 인덱싱
  - `parse/custom_type_graph.rs` — 커스텀 타입 재귀 탐색
- [ ] `schema/types.rs` — 기존과 같은 골격 + `MetaScoped`, `FixedArrayElement`, `Anonymous` variant
- [ ] `schema/error.rs`, `constants.rs` 신규 작성
- [ ] `Cargo.toml` 의존성: `clap`, `csv`, `thiserror`, `serde`, `serde_json`
- [ ] `tests/fixtures/`에 ClassJob.csv / Recipe.csv 샘플 (각 5행)
- [ ] CLI: `gleaner schema --input <CSV> --output <.d.ts>` 동작 (단일 CSV 디버깅용)

#### 완료 기준

- `cargo test`에서 fixture 기반 단위 테스트 모두 통과
- `gleaner schema`로 ClassJob.csv 1개를 처리해 정상 `.d.ts` 출력
- 필드명 정규화 결과가 [[feature-scope]] § 5의 표와 일치 (예: `Item{Result}` → `itemResult`)

#### 예상 작업량

3 ~ 5일.

### 5.2 D-3 — Gleaner Layer 2: Domain Enricher

**목표**: 여러 CSV를 조합해 `domain::Item`, `domain::Recipe`, `domain::ClassJob` 객체를 만들 수 있다.

**대응 마일스톤**: M3 + M4

#### 산출물

- [ ] `enrich/` 모듈 (8개 파일):
  - `normalize.rs` — 익명 드롭, 일반 정규화 보강
  - `array.rs` — 고정 배열 collapse (Recipe ingredient 8슬롯 → 가변)
  - `base_param.rs` — BaseParam 이름 embed
  - `recipe_level.rs` — RecipeLevelTable flatten
  - `optional.rs` — 데이터 기반 optional 판정 + override 로드
  - `item.rs` / `recipe.rs` / `class_job.rs` — 도메인 조립 (raw 패스 스루)
- [ ] `domain/` 모듈 (4개 파일): `item.rs`, `recipe.rs`, `class_job.rs`, `base_param.rs`
- [ ] `overrides/optional.toml` 골격 + 1~2개 샘플 항목
- [ ] `reports/optional-analysis.md` 1차 출력 (사람이 한 번 검토)

#### 완료 기준

- Recipe 1행을 처리해 `domain::Recipe`에 `recipeLevel`/`maxQuality`가 정확히 주입됨
- Recipe의 8슬롯 ingredient에 id 0이 섞인 케이스에서 양쪽 배열이 같은 인덱스로 동기화 드롭됨
- Item의 `baseParam0..5` 슬롯이 모두 `{ name, value }` 객체로 해석됨
- optional 판정 임계치(기본 30%)가 코드에 명시돼 있고 override 파일이 적용됨

#### 예상 작업량

5 ~ 7일.

### 5.3 D-4 — Gleaner Layer 3 + 전체 파이프라인

**목표**: `gleaner extract`로 모든 도메인의 JSON 산출물이 생성된다.

**대응 마일스톤**: M5 + M6

#### 산출물

- [ ] `serialize/` 모듈 (7개 파일): `item.rs`, `recipe.rs`, `class_job.rs`, `base_param.rs`, `item_ui_category.rs`, `writer.rs`, `mod.rs`
- [ ] `cli.rs` 서브커맨드 구조 구현 (`extract` / `schema` / `diff`)
- [ ] `dist/data/`에 [[data-contract]] § 1의 모든 파일 출력:
  - `itemSummaries.json` (~2MB)
  - `item/{id}.json` × 약 38,000
  - `recipeSummaries.json` (~200KB)
  - `recipe/{id}.json` × 약 4,000
  - `classJobs.json`, `baseParams.json`, `itemUICategories.json`

#### 완료 기준

- `gleaner extract --raw-dir <ff14-raw-data> --out-dir ./dist`가 0 에러로 종료
- 출력 파일 크기/개수가 [[data-contract]] § 1 예상치와 ±20% 이내 일치
- `id` 오름차순 정렬, 결정적 들여쓰기 등 [[gleaner-redesign]] § 6.2 결정성 조건 충족
- ClassJob 단일 파일을 사람이 한 번 열어 [[data-contract]] § 6.3과 비교 검증

#### 예상 작업량

5 ~ 7일.

### 5.4 D-5 — TypeScript 출력 + Schema Diff

**목표**: Frontend가 import할 수 있는 `.d.ts`가 생성되고, 이전 빌드와의 변경이 리포트된다.

**대응 마일스톤**: M7 + M8

#### 산출물

- [ ] `serialize/typescript.rs` — 도메인 객체 기준 .d.ts 출력
  - 기존 `web-projects` 코드의 `schema/typescript.rs`(291줄) 로직을 reference로 활용 가능
  - 단, 출력 대상은 *raw schema*가 아니라 *Layer 2 enriched 도메인*
- [ ] `serialize/diff.rs` — 이전 빌드와 .d.ts 비교
- [ ] `gleaner diff` 서브커맨드 단독 실행
- [ ] `dist/types/noumenon.d.ts` 출력
- [ ] `dist/reports/schema-diff.md` 출력 (첫 빌드는 "신규 생성" 표시)
- [ ] (선택) `rayon` 도입 여부 판단 — 38k 파일 직렬 쓰기 실측 후 필요 시
- [ ] 결정성 테스트: `extract` 2회 연속 실행 → 산출물 diff 0

#### 완료 기준

- `noumenon.d.ts`가 [[data-contract]] § 6의 엔티티 메모와 의도적으로 일치
- `schema-diff.md`가 두 빌드 사이의 변화를 사람이 읽을 수 있는 형태로 표현
- 같은 raw 입력으로 2회 실행 시 산출물이 byte-identical

#### 예상 작업량

3 ~ 5일.

> 💡 **인사이트 — 왜 D-5를 별도 milestone으로 분리하는가**
> 
> .d.ts 출력은 D-4의 부산물처럼 보이지만, **Frontend가 시작될 수 있는 유일한 진입점**이라는 의미에서 분리할 가치가 있습니다. D-4 끝에서 *"JSON은 나오지만 .d.ts가 없다"*면 Frontend는 한 줄도 시작할 수 없습니다. Schema diff까지 포함한 D-5가 끝나면 *"Gleaner는 신뢰할 만한 외부 의존성"*이 되고, 이후 Frontend는 Gleaner 코드를 다시 열지 않고도 작업할 수 있게 됩니다.

---

## 6. Frontend 단계 (D-6 ~ D-8)

[[frontend-plan]]의 § 6.1 MUST 항목들을 3단계로 그룹화.

### 6.1 D-6 — Frontend 골격

**목표**: Vite + React 19 + Router v7 + React Query + Zustand 골격이 동작하고, `@noumenon/types`로 .d.ts를 import할 수 있다.

#### 산출물

- [ ] `apps/front/` 신규 Vite + React 19 프로젝트
- [ ] `tsconfig.json`에 paths alias:
  ```jsonc
  "paths": {
    "@noumenon/types": ["../../packages/gleaner/dist/types/noumenon"]
  }
  ```
- [ ] `src/main.tsx`, `App.tsx`, `router.tsx`
- [ ] React Router v7 라우트 골격: `/`, `/search`, `/item/:id`, `/recipe/:id`, `/classjob`, `/classjob/:id`, `*`
- [ ] Root loader: `classJobs.json`, `baseParams.json`, `itemUICategories.json` 프리페치 + Map 변환
- [ ] `src/lib/lookups.ts` (Map 빌드 헬퍼)
- [ ] `src/lib/decoder.ts` (gubal `utils/decoder.ts` 로직 계승)
- [ ] `src/styles/tokens.css` (CSS variables, 다크/라이트)
- [ ] `NotFoundPage`, `HomePage` 더미 구현

#### 완료 기준

- `bun dev`로 개발 서버 기동, `/`에서 root loader가 `classJobs.json` 로드 성공
- `import type { Item } from '@noumenon/types'`가 컴파일 에러 없음
- `bun build`로 정적 산출물 생성 성공

#### 예상 작업량

3 ~ 4일.

### 6.2 D-7 — Search + Item 상세

**목표**: 검색 페이지에서 Item을 찾고, 상세 페이지에서 FK가 클릭 가능한 링크로 표시된다.

#### 산출물

- [ ] `SearchPage`:
  - `itemSummaries.json` lazy load (route loader)
  - `SearchInput` (controlled, 300ms 디바운스, IME 처리)
  - `SearchResult` (`@tanstack/react-virtual` 가상화 리스트)
  - `ItemInline` 행 (아이콘 + 이름 + 레어도 색상)
  - URL 딥링크: `?q=`, `?category=`, `?sort=`
- [ ] `ItemDetailPage`:
  - `item/{id}.json` route loader 페치
  - `ItemDetail` (탭 구조: 개요/장착/제작/환상)
  - `ItemIcon`, `ItemName`, `BonusParameters`, `MateriaSlots`, `CraftingAndRepair`, `StorageInformation`
  - **FK 링크**: `repairItem`, `glamourItem`, `useClassJob`, `repairClassJob` 클릭 시 해당 엔티티로 이동
- [ ] `src/labels/item.ts` — 표시 라벨 맵 (예: `physicalDamage` → `'물리 공격력'`)

#### 완료 기준

- 38k 아이템 검색이 60fps 유지
- "단소 검 → 상세 → repairItem 클릭 → 다른 아이템 상세"가 URL 변화와 함께 동작
- 뒤로가기 1회로 이전 화면 복원

#### 예상 작업량

5 ~ 7일.

### 6.3 D-8 — Recipe / ClassJob 상세 + UX MUST 마무리

**목표**: 모든 엔티티 페이지가 동작하고, [[frontend-plan]] § 6.1 MUST 항목이 전부 반영됐다.

#### 산출물

- [ ] `RecipeDetailPage`, `RecipeInline`, `RecipeDetail`, `IngredientList`
- [ ] `ClassJobListPage` (19개 일람), `ClassJobDetailPage`
- [ ] `ClassJobBadge` (FK 해석된 자리에서 재사용)
- [ ] **다크 모드**: `ThemeToggle` + `prefers-color-scheme`
- [ ] **키보드 내비**: `/`로 검색 포커스, `esc`/`↑↓`/`enter`
- [ ] **모바일 레이아웃**: <768px 1-column, 탭 아코디언화
- [ ] **아이콘 lazy loading**: `<img loading="lazy">`
- [ ] `src/labels/recipe.ts`, `src/labels/classJob.ts`

#### 완료 기준

- [[frontend-plan]] § 6.1의 6개 MUST 항목 전부 동작
- 모바일 (iPhone 13 Safari) / 데스크톱 (Chrome) 양쪽에서 사용 가능
- Lighthouse 접근성 점수 90+

#### 예상 작업량

5 ~ 7일.

---

## 7. D-9 — 배포 + 외부 공개

**목표**: `noumenon.<도메인>`에서 외부 사용자가 접속할 수 있다.

### 산출물

- [ ] **CDN 결정**: gubal S3 버킷 재사용 vs Cloudflare R2 신규 (의사결정 별도)
- [ ] JSON 산출물 CDN 업로드 (Gleaner 빌드 후 업로드 스크립트)
- [ ] **배포 타깃 결정**: Cloudflare Pages 권장 (무료 한도 여유)
- [ ] `apps/front/` 정적 빌드 → 배포 파이프라인
- [ ] 도메인 연결 (사용자 보유 도메인 또는 `*.pages.dev` 임시)
- [ ] README 외부용 다듬기:
  - 라이브 데모 링크
  - "어떤 데이터를 보여주는가"의 사용자 친화적 설명
  - 기여 가이드 (issue 우선)
  - 라이선스 (MIT 권장)
- [ ] (선택) 첫 release 태그 `v0.1.0` + GitHub Release notes

### 완료 기준

- 외부 사용자가 URL 1개로 접속해 검색 → 상세 진입 가능
- 모바일 데이터 환경에서 초기 로드 < 3s
- README만 보고 협력자가 로컬 개발 환경을 구축할 수 있다

### 예상 작업량

2 ~ 3일.

> 💡 **인사이트 — 왜 D-9에서 README를 한 번 더 다듬는가**
> 
> D-1에서 작성한 README는 *내가 다시 보기 위한* 문서입니다. D-9 시점의 README는 *처음 들어온 외부인이 5초 안에 "이게 뭔지" 파악하는* 문서가 돼야 합니다. 두 목적은 비슷해 보이지만 다른 글입니다 — *내가 아는 것을 모두 적은 글*은 외부인에게 정보 과부하가 되고, *외부인에게 충분한 글*은 내가 보기엔 정보 부족이 됩니다. 단계 분리가 자연스러운 이유.

---

## 8. D-10 — NICE 기능 (보류)

[[frontend-plan]] § 6.2의 NICE 항목들. 외부 공개 후 사용 피드백을 받고 우선순위 재조정.

| 항목 | 비고 |
|---|---|
| 초성 검색 | gubal README의 미구현 계획. ~50줄 함수 |
| 히스토리 breadcrumb | in-app 탐색 경로 표시 |
| 카테고리 트리 네비 | `ItemUICategory.major/minorOrder` 활용 |
| Recipe 역탐색 | "이 재료가 쓰이는 레시피" 클라이언트 인덱스 |
| 비교 뷰 | 2~3개 Item 나란히 |

이 단계는 구체 일정을 두지 않고, 사용 빈도가 높아 보이는 것부터 issue로 등록.

---

## 9. 위험 및 대응

| 위험 | 발생 시점 | 영향 | 대응 |
|---|---|---|---|
| **`exd-header.json` 포맷이 예상보다 복잡** | D-2 | Layer 1 일정 +1주 | D-0에서 한 번 훑어 사전 파악. 복잡하면 별도 캐시 모듈 분리 |
| **optional 판정 임계치(30%) 부적합** | D-3 | override 파일 비대화 | M4 후 실측, 임계치 조정. override가 50줄을 넘으면 알고리즘 자체 재검토 |
| **38k 개별 파일 쓰기 성능** | D-4 | extract 시간 5분+ | rayon 도입 (의존성 1개). 그래도 부족하면 sharded 출력 검토 |
| **2MB itemSummaries 모바일 로드 지연** | D-7 / D-9 | 검색 진입 UX 저하 | (a) gzip/brotli 검증, (b) Gleaner의 압축 옵션(객체 배열 → 숫자 배열) 활성화 |
| **Cloudflare Pages 무료 한도 초과** | D-9 이후 | 추가 비용 발생 | 트래픽 모니터링, 한도 근접 시 CDN 분리 또는 plan upgrade |
| **raw-data 갱신 주기와 배포 주기 불일치** | D-9 이후 | 데이터 stale | Gleaner extract → CDN 업로드를 반자동화 (수동 트리거 + GitHub Actions) |

---

## 10. acceptance criteria 통합 표

각 단계가 *진짜로 끝났는지* 한눈에 보기 위한 체크포인트.

| 단계 | 한 줄 완료 기준 |
|---|---|
| D-0 | 신규 레포 시작 시 *기존 코드를 다시 열 필요가 없다* |
| D-1 | `bun install && turbo run check-types`가 신규 레포에서 통과, web-projects가 깨지지 않음 |
| D-2 | `gleaner schema`로 단일 CSV → .d.ts 정상 생성 |
| D-3 | `domain::Recipe`에 `recipeLevel`/`maxQuality` 정확 주입, ingredient 동기화 드롭 |
| D-4 | `gleaner extract`로 [[data-contract]] § 1의 모든 JSON 출력 |
| D-5 | `noumenon.d.ts` 생성 + 결정적 (2회 실행 byte-identical) |
| D-6 | `import type { Item } from '@noumenon/types'` 컴파일 + root loader 동작 |
| D-7 | 검색 → Item 상세 → FK 클릭 → 다른 Item 동선이 URL과 함께 동작 |
| D-8 | [[frontend-plan]] § 6.1 MUST 6개 전부 동작 |
| D-9 | 외부 URL 1개로 접속 가능, README 5초 파악 |
| D-10 | (보류) |

---

## 11. 다음 단계

1. **사용자 검토**: 이 roadmap의 단계 분할/완료 기준 합의
2. **D-0 착수**: 기존 `web-projects/packages/noumenon-gleaner/src/` 코너케이스 추출 (반나절)
3. **dotori 측 작업**: `noumenon.md` 프로젝트 허브에 Phase C/D 완료 로그 추가
4. **D-1 착수**: `siluat/noumenon` GitHub 레포 생성

이 문서는 D-1에서 신규 레포 `docs/roadmap.md`로 이관된다. 이관 후에도 dotori의 마스터 사본은 작성 이력 보존용으로 유지하되, 진행 상태 갱신은 신규 레포 쪽만 반영한다.

> 💡 **인사이트 — roadmap의 일정 추정에 대해**
> 
> 각 단계의 "예상 작업량"은 *연속 작업 가정*의 조잡한 추정입니다. 실제 진행에서는 컨텍스트 스위칭, raw 데이터의 예상 못한 기벽, *"이 모듈을 잠깐 더 깔끔하게 짜고 싶다"*는 충동 등으로 1.5~2배 늘어나는 게 정상입니다. 합산해 *최소 한 달, 최대 두 달*의 1인 프로젝트 규모로 봐도 무방합니다. 일정 압박 없는 학습 프로젝트라는 점에서, *"마감"보다 *"각 단계의 완료 기준"*에 충실한 게 낫습니다.
