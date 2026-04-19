# Gleaner Redesign (Phase C)

#unreviewed

Noumenon의 데이터 추출 축(`packages/gleaner`) 설계 문서. raw-data가 스키마의 single source of truth라는 원칙(=[[feature-scope]] 원칙 1)을 코드 구조로 구현한다.

- 상위: [[noumenon]]
- 선행: [[feature-scope]], [[data-contract]]

---

## 0. 설계 원칙

이 문서의 모든 결정은 다음 4개 원칙에 종속된다. 이 원칙과 충돌하는 설계는 채택하지 않는다.

1. **raw-data가 스키마의 단일 진실**. Gleaner는 raw → 코드/JSON으로 흐르는 **단방향** 변환기.
2. **Gleaner는 영원히 생성자**. "검증자로 전환"은 없음. 타입(`.d.ts`)도 항상 Gleaner 생성물.
3. **결정론적 알고리즘만으로 변환**. 사람이 손으로 관리하는 "필드명 화이트리스트/오버라이드 테이블"은 데이터 계약의 일부가 될 수 없음. 표시 라벨 같은 사람 취향은 Frontend 표시 라벨 맵으로 분리.
4. **raw 변화는 silent divergence 없이 표면화**. Gleaner는 매 빌드마다 이전 빌드와의 스키마 diff를 리포트.

---

## 1. 배경: 현 구현의 재실측 (2026-04-16)

| 파일 | 줄 수 | 책임 |
|---|---|---|
| `src/main.rs` | 95 | entry, CLI 파싱, 에러 출력 분기 |
| `src/cli.rs` | 18 | clap 기반 `--input-file-path` / `--output-file-path` |
| `src/constants.rs` | 17 | `BASIC_TYPES`, `CUSTOM_TYPE_PATTERNS`, `SPECIAL_TYPES`, CSV 헤더 상수 |
| `src/schema/mod.rs` | 11 | 재-export |
| `src/schema/types.rs` | 124 | `FieldType` 열거형, `Field`, `Schema`, `SchemaMap` |
| `src/schema/builder.rs` | **710** | `SchemaBuilder`: CSV 파싱 + 타입 추론 + 커스텀 타입 재귀 탐색 + 순환 의존성 감지 |
| `src/schema/typescript.rs` | 291 | `TypeScriptGenerator`: interface 생성 |
| `src/schema/error.rs` | 25 | `SchemaError` (thiserror) |
| `src/schema/utils.rs` | 223 | `analyze_missing_files` 등 |
| **합계** | **1,514** | |

### 의존성 (`Cargo.toml`)
```toml
[dependencies]
clap = { version = "4.5.40", features = ["derive"] }
csv = "1.3"
thiserror = "1.0"
```

### 현 구현이 **이미 할 수 있는** 것
- 3행 헤더 CSV 파싱 (`key+index` / `# + fieldname` / `typename`)
- FF14 raw 타입 분류: `str / int32 / uint32 / int16 / uint16 / byte / sbyte / float / bool / bit&XX / Image / Row / Key / Color / Custom(...)`
- 커스텀 타입 참조 재귀 탐색, 순환 의존성 감지
- TypeScript interface 파일 1개 생성
- 누락 CSV 진단 메시지

### 현 구현에 **없는** 것 (재설계 범위)
1. **실제 행 데이터 변환** (CSV row → JSON value). 현재는 스키마(메타데이터)만 추출.
2. **FF14 메타 문법 정규화 알고리즘**: `Item{Result}` → `itemResult`, `Item{Ingredient}[0..7]` → `itemIngredients[]`.
3. **도메인 보강**: `RecipeLevelTable` flatten, `BaseParam` 이름 embed.
4. **JSON 직렬화** (`serde` / `serde_json` 의존성 없음).
5. **다수 CSV → 다수 JSON 군** 파이프라인 CLI.
6. **이전 빌드 대비 스키마 diff 리포트** (원칙 4의 실체).
7. **데이터 기반 optional 판정** (전수 스캔 + 임계치).

---

## 2. 재설계 원칙: 3-Layer 아키텍처

현재 `builder.rs`(710줄)에 뭉쳐 있는 책임을 3개 레이어로 분리.

```
  rawexd/*.csv
       │
       ▼
┌────────────────────────────────────────────────┐
│ Layer 1: Pure Parser                           │
│  - 단일 CSV만 본다                              │
│  - CSV 3행 헤더 → Schema                        │
│  - 행 데이터 → 구조화된 Rust 값 (raw DOM)        │
│  - 결정론적 알고리즘으로 필드명 정규화            │
│  - 순수함수: 동일 입력 → 동일 출력               │
│  - 바깥 세계(파일시스템, 다른 CSV) 모름          │
└──────────────────┬─────────────────────────────┘
                   │ Vec<RawRecord>, Schema
                   ▼
┌────────────────────────────────────────────────┐
│ Layer 2: Domain Enricher                       │
│  - 여러 CSV를 조합한다                          │
│  - FK 해석 (BaseParam 이름 embed)               │
│  - RecipeLevelTable flatten                    │
│  - 익명 필드 드롭, 타입 교정 (exd-header.json)    │
│  - 데이터 기반 optional 판정 + 오버라이드         │
│  - **수동 어댑터 없음**                        │
│  - 결과: 도메인 객체 (raw 모든 필드 패스 스루)    │
└──────────────────┬─────────────────────────────┘
                   │ DomainModel (Item, Recipe, ClassJob, ...)
                   ▼
┌────────────────────────────────────────────────┐
│ Layer 3: Serializer                            │
│  - 도메인 객체 → JSON 파일 군                    │
│  - itemSummaries.json, item/{id}.json, ...     │
│  - TypeScript .d.ts 생성 (single source of     │
│    truth로서의 타입 출력 — § 6 참조)            │
│  - 이전 빌드와 diff 리포트                       │
└────────────────────────────────────────────────┘
                   │
                   ▼
  dist/
  ├── data/
  │   ├── itemSummaries.json
  │   ├── item/{id}.json × 38,000
  │   ├── recipeSummaries.json
  │   ├── recipe/{id}.json × 4,000
  │   ├── classJobs.json
  │   ├── baseParams.json
  │   └── itemUICategories.json
  ├── types/noumenon.d.ts
  └── reports/schema-diff.md      ← 이전 빌드 대비 변경
```

### 설계 근거

- **테스트 가능성**: Layer 1은 순수함수로 쪼개면 Recipe.csv 샘플 한 줄만으로 단위 테스트 가능. Layer 2는 fixture 기반 골든 테스트, Layer 3은 파일시스템 mock으로 분리.
- **책임 경계**: "한 CSV만 보는 로직"과 "CSV들을 합쳐 의미를 만드는 로직"이 섞이면 710줄 빌더가 다시 만들어짐. 분리가 곧 유지보수.
- **확장성**: 나중에 Action/Quest 엔티티를 추가할 때 Layer 2의 새 모듈(`enrich/action.rs`) 하나만 추가하면 됨. Layer 1/3은 건드리지 않는다.
- **단방향 흐름의 강제**: 원칙 1을 코드 구조로 못박는 장치. raw → Layer 1 → Layer 2 → Layer 3 → 파일. 거꾸로 흐르는 호출(예: Frontend types를 읽어 Gleaner 동작 변경)은 컴파일 시점에 불가능.

### 순수함수 원칙

Layer 1과 Layer 2는 **파일시스템과 분리**. 파일 IO는 Layer 3(출력)과 CLI(입력)만 담당. 테스트에서 `CsvData` 구조체에 직접 문자열을 넣어 검증 가능.

---

## 3. 계승 vs 신규 매핑

| 현 모듈 | 재설계 처분 | 비고 |
|---|---|---|
| `schema/types.rs` (`FieldType`, `Schema`) | **계승** + 확장 | `Custom(String)` 옆에 `MetaScoped`, `FixedArrayElement`, `Anonymous` 등 raw 헤더 메타 표현 추가 |
| `schema/builder.rs` (SchemaBuilder) | **일부 계승 + 분할** | CSV 헤더 파싱/타입 추론 → Layer 1. 커스텀 타입 재귀 탐색은 Layer 2로 이동 |
| `schema/typescript.rs` | **계승** + 출력 대상 확장 | data-contract의 interface가 아니라 **현 raw 스냅샷 전부**를 .d.ts로 출력 (원칙 2) |
| `schema/error.rs` | **계승** + 확장 | `EnrichError`, `SerializeError`, `DiffError` 등 추가 |
| `schema/utils.rs` (누락 파일 진단) | **계승** | 사용자 친화적 에러 메시지는 그대로 유지 |
| `constants.rs` | **계승** + 소폭 확장 | `ANONYMOUS_FIELD_MARKER` 등. **`FIELD_NAME_OVERRIDES` 같은 매핑 테이블은 추가하지 않음** (원칙 3) |
| `cli.rs` | **교체** | `gleaner extract` / `gleaner schema` / `gleaner diff` 서브커맨드 (§ 7) |

**새로 추가할 의존성**
```toml
serde = { version = "1", features = ["derive"] }
serde_json = "1"
# 필요 시:
# indexmap = "2"   # HashMap 순서 고정 (스키마 결정성)
# similar = "2"    # diff 출력 (Markdown 형태)
# rayon = "1"      # 38k 파일 쓰기 병렬화 (실측 후)
```

---

## 4. Layer 1: Pure Parser 상세 설계

### 4.1 입력/출력

**입력**: 단일 CSV 파일 경로.  
**출력**: `CsvData` 구조체 (스키마 + 원시 행들).

```rust
pub struct CsvData {
    pub schema: Schema,              // 기존 Schema 재활용 + FieldType 변종 확장
    pub records: Vec<RawRecord>,     // 신규
}

pub struct RawRecord {
    pub key: i64,                    // CSV 첫 컬럼 (key)
    pub values: Vec<RawValue>,       // 스키마 필드 순서와 동일
}

pub enum RawValue {
    String(String),
    Int(i64),
    Uint(u64),
    Float(f64),
    Bool(bool),
    Bits(u32),                       // bit&XX 원본 값
    Null,                            // 빈 문자열 처리
}
```

### 4.2 신규 추가할 타입 분류

기존 `FieldType`에 FF14 메타 문법 대응 variant 추가:

```rust
pub enum FieldType {
    // ... (기존 변종 유지)

    /// Raw CSV 헤더가 "Item{Result}" 같은 형태
    /// 알고리즘으로 필드명만 풀어주고 타입은 Custom(base)로 처리
    MetaScoped {
        base: String,                // "Item"
        scope: String,               // "Result"
    },

    /// "Item{Ingredient}" 헤더가 인덱스 [0..N]으로 반복되는 경우
    /// Layer 1은 개별 컬럼으로 보되, Layer 2가 배열로 collapse
    FixedArrayElement {
        base: String,                // "Item"
        scope: Option<String>,       // "Ingredient" 또는 None (BaseParam[0..5])
        index: u8,
    },

    /// SaintCoinach가 헤더를 풀지 못한 익명 컬럼
    Anonymous,
}
```

Layer 1은 **해석하지 않고** 형태만 인식한다. "이 8개가 모여서 배열이 된다"는 의미 부여는 Layer 2의 책임.

### 4.3 필드명 정규화 (결정론적 알고리즘)

[[feature-scope]] § 5의 표를 그대로 코드로:

```rust
pub fn normalize_field_name(raw: &str) -> String {
    // 우선순위 (모두 결정론적, 데이터 의존 없음):
    // 1. X{Y}[N] → 배열 요소 (인덱스 분리해서 base 부분만 normalize)
    // 2. X{Y}    → camelCase 결합 (xY)
    // 3. X[N]    → 배열 또는 슬롯 유지 (BaseParam[0..5]는 슬롯 유지 — § 5.1)
    // 4. PascalCase → camelCase
    // 5. 그 외   → 그대로
}
```

**오버라이드 테이블 없음**. `FIELD_NAME_OVERRIDES` / `ITEM_FIELD_MAP` 등 사람이 손으로 관리하는 매핑을 두지 않는다 (원칙 3). 사람 손맛 축약 라벨(`physDamage`, `modifierHp` 등)이 UI에 필요하면 Frontend의 표시 라벨 맵에서 처리.

### 4.4 `exd-header.json` 활용

`/Users/siluat/Projects/ff14-raw-data/exd-header.json` (2.6MB, 7,777 테이블)을 Layer 1에서 로드해 타입/이름 보정기로 사용.

**교정 사례**
- `ClassJob.Name{English}`의 CSV 타입 행은 `byte`이지만 실제 데이터는 문자열. `exd-header.json` 참조해 `FieldType::String`으로 강제.
- 익명 컬럼이 `exd-header.json`에 이름이 있다면 복원해 Anonymous 대신 정상 필드로.

```rust
pub struct ExdHeaderIndex { /* JSON 파싱 결과 */ }
impl ExdHeaderIndex {
    pub fn correct_type(&self, table: &str, field: &str, csv_type: &FieldType) -> FieldType { ... }
    pub fn restore_name(&self, table: &str, column_index: u32) -> Option<String> { ... }
}
```

CLI 레벨에서 1회 로드하고 Layer 1 함수에 참조로 주입.

### 4.5 단위 테스트 전략

기존 `builder.rs`의 테스트 패턴 유지. 각 타입 분기마다 최소 샘플 CSV 문자열을 인라인 fixture로.

---

## 5. Layer 2: Domain Enricher 상세 설계

### 5.1 책임 (수동 어댑터 없음)

| 책임 | 구현 모듈 (제안) |
|---|---|
| 고정 배열 `[0..N]` 컬럼들 → 가변 배열 collapse | `enrich/array.rs` |
| 익명 필드 드롭 (Layer 1이 표시한 것 제외) | `enrich/normalize.rs` |
| `BaseParam` 이름 embed (id → `{ name, value }`) | `enrich/base_param.rs` |
| `RecipeLevelTable` flatten (`recipeLevel`, `maxQuality`) | `enrich/recipe_level.rs` |
| 데이터 기반 optional 판정 + 오버라이드 적용 | `enrich/optional.rs` |
| 각 엔티티의 Domain 객체 조립 (raw 패스 스루) | `enrich/item.rs` / `recipe.rs` / `class_job.rs` |

**의도적으로 빠진 것**: 사람 손으로 만든 필드명 어댑터. 원칙 3에 따라 `enrich/manual_adapter.rs` 같은 모듈은 만들지 않는다. 알고리즘 변환 결과 그대로 도메인 필드명이 됨.

### 5.2 배열 collapse

입력: 동일 base/scope를 가진 `FixedArrayElement` 컬럼 N개.  
출력: 1개의 가변 배열 필드 (혹은 슬롯 유지).

```rust
pub fn collapse_fixed_arrays(record: &RawRecord, schema: &Schema) -> EnrichedRecord {
    // Recipe.csv 예시:
    //   Item{Ingredient}[0] = 5057  →  itemIngredient = [5057, 5058, ...]
    //   Item{Ingredient}[1] = 5058
    //   Item{Ingredient}[2] = 0      (drop: id가 0이면 빈 슬롯)
    //   ...
    //   Amount{Ingredient}[0] = 3    →  amountIngredient = [3, 1, ...]
}
```

**핵심 규칙 ([[feature-scope]] § 2 근거)**
- Ingredient 배열은 `id == 0`인 슬롯을 **양쪽에서 같은 인덱스로 동시 드롭**해 인덱스 동기화 유지.
- `BaseParam[0..5]`는 슬롯 유지 (`baseParam0..5` 개별 필드). 이는 [[feature-scope]] § 5.1의 명시적 정책.

### 5.3 `RecipeLevelTable` flatten

Recipe의 `RecipeLevelTable` FK를 따라가 `recipeLevel`(숫자)과 `maxQuality`(정수)를 Recipe 도메인 객체에 직접 주입.

```rust
pub struct RecipeLevelLookup {
    table: HashMap<i64, RecipeLevelRow>,
}
impl RecipeLevelLookup {
    pub fn load(raw_dir: &Path) -> Result<Self, EnrichError> { ... }
    pub fn flatten_into_recipe(&self, recipe: &mut Recipe, ref_id: i64) {
        let row = self.table.get(&ref_id);
        recipe.recipe_level = row.map(|r| r.class_job_level).unwrap_or(0);
        recipe.max_quality = row.map(|r| r.quality).unwrap_or(0);
    }
}
```

Frontend는 `RecipeLevelTable`이 존재했다는 사실 자체를 모른다. **Gleaner에서 지식이 소모**된다.

### 5.4 `BaseParam` 이름 embed

Item의 `baseParam0..5`는 `{ name: string; value: number }` 객체. 이는 알고리즘 정규화로 자동 도출되지 않는 **도메인 변환**이므로 Layer 2의 명시적 책임.

```rust
pub struct BaseParamLookup {
    names: HashMap<i64, String>,
}
impl BaseParamLookup {
    pub fn resolve(&self, id: i64, value: i64) -> Option<BaseParam> {
        if id == 0 { return None; }
        Some(BaseParam {
            name: self.names.get(&id)?.clone(),
            value,
        })
    }
}
```

추가로 `baseParams.json`(id + name)을 별도 출력 — Frontend가 `ClassJob.primaryStat`(BaseParam id 단독 참조)를 해석할 때 사용.

### 5.5 데이터 기반 optional 판정 + 오버라이드 (재설계)

기존 안의 `OPTIONAL_FIELDS` 하드코딩은 **원칙 1과 충돌**해 폐기. 다음 하이브리드로 대체:

```rust
pub struct OptionalAnalysis {
    pub field_name: String,
    pub null_or_zero_ratio: f64,    // 0.0 ~ 1.0
    pub decided_optional: bool,
}

pub fn analyze_optional(records: &[RawRecord], schema: &Schema, threshold: f64) -> Vec<OptionalAnalysis> {
    // 컬럼별로 (Null + (id 타입 컬럼이면 0) 비율) 계산
    // threshold(예: 0.30) 이상이면 optional 후보
}
```

판정 결과는:
1. **`reports/optional-analysis.md`**로 출력 (사람이 한 번 검토).
2. 명시적 오버라이드는 **별도 파일**(`overrides/optional.toml`)로 분리:
   ```toml
   # 데이터로는 optional이지만 의미상 항상 있는 것으로 강제
   [[force_required]]
   entity = "Item"
   field  = "name"

   # 데이터로는 항상 있지만 의미상 optional로 표시
   [[force_optional]]
   entity = "Item"
   field  = "salvage"
   ```
3. 코드에 하드코딩하지 않음. 오버라이드 파일도 git에 남아 변경 이력이 추적되며, 새 raw 변화 시 갱신 필요성이 리포트로 표면화됨.

---

## 6. Layer 3: Serializer 상세 설계

### 6.1 출력 산출물

| 파일 | 크기 (예상) | 담당 생성기 |
|---|---|---|
| `data/itemSummaries.json` | ~2MB | `serialize/item.rs::write_summaries` |
| `data/item/{id}.json` × 38k | ~2KB each | `serialize/item.rs::write_details` |
| `data/recipeSummaries.json` | ~200KB | `serialize/recipe.rs::write_summaries` |
| `data/recipe/{id}.json` × 4k | ~500B each | `serialize/recipe.rs::write_details` |
| `data/classJobs.json` | ~5KB | `serialize/class_job.rs::write_all` |
| `data/baseParams.json` | ~1KB | `serialize/base_param.rs::write_all` |
| `data/itemUICategories.json` | ~10KB (선택) | `serialize/item_ui_category.rs::write_all` |
| `types/noumenon.d.ts` | 수 KB | `serialize/typescript.rs` |
| `reports/schema-diff.md` | 가변 | `serialize/diff.rs` |
| `reports/optional-analysis.md` | 가변 | `serialize/diff.rs` |

### 6.2 결정성 (determinism)

동일 입력 → 동일 파일. 빌드 산출물 diff 검증 가능.

- 엔티티 배열은 항상 `id` 오름차순 정렬 후 쓰기.
- `serde_json::to_writer_pretty`로 들여쓰기 고정.
- HashMap 순회 순서에 의존하지 않도록 `indexmap::IndexMap` 또는 key 정렬 후 직렬화.

### 6.3 개별 상세 파일 쓰기 효율

`item/{id}.json` 38,000개를 단일 스레드로 쓰면 수 분. `rayon` 도입 검토 (의존성 추가 전 실측 후 결정).

### 6.4 TypeScript 생성: 항상 생성자, 항상 단일 진실

기존 `schema/typescript.rs` (291줄)를 유지하되, 출력 대상을 바꾼다:

- **Before (구 버전)**: 입력 CSV의 `Schema`를 그대로 `interface`로 변환.
- **After**: Layer 2의 도메인 객체(엔티티별로 `Item`, `Recipe`, `ClassJob`, 보조 `ItemSummary`, `RecipeSummary`, `BaseParam`, `ItemUICategory`)를 .d.ts로 출력. 이는 raw + Layer 2 enrich 결과의 **결정론적 표현**이며 Frontend의 import 대상.

**"검증자 모드"는 폐기**. 원칙 2에 따라 Gleaner는 항상 생성자다. data-contract.md 같은 사람 문서와 .d.ts를 비교하는 모드는 만들지 않는다 (사람 문서 쪽이 어긋났다면 사람 문서를 갱신할 일이지, 코드가 검증할 일이 아님).

대신 다음을 출력:

- **`reports/schema-diff.md`**: 이전 빌드와 .d.ts diff. 새 필드/사라진 필드/타입 변경을 명시. CI에서 이 파일이 변하면 PR 리뷰에 자동 표시.

이게 원칙 4의 실체화. raw 변화가 코드 산출물 변화로 자동 전파되며, 사람은 diff만 검토하면 됨.

---

## 7. CLI 재설계

### 7.1 현 CLI

```
gleaner --input-file-path <CSV> --output-file-path <.d.ts>
```

단일 CSV에 대한 스키마 추출 1회 실행.

### 7.2 서브커맨드 구조

```
gleaner extract \
  --raw-dir /Users/siluat/Projects/ff14-raw-data \
  --out-dir  ./dist \
  [--overrides ./overrides/optional.toml] \
  [--prev-snapshot ./dist-prev]            # diff 기준점 (CI에서는 main 브랜치 빌드)

gleaner schema \
  --input <CSV> \
  --output <.d.ts>                         # 단일 CSV 디버깅용 (기존 호환)

gleaner diff \
  --current ./dist/types/noumenon.d.ts \
  --previous ./dist-prev/types/noumenon.d.ts \
  --output ./dist/reports/schema-diff.md   # extract 호출 시 자동 실행되지만 단독 실행도 가능
```

- `extract`: 메인 파이프라인 (Layer 1→2→3 전체 + diff 자동 출력).
- `schema`: 기존 사용법 보존. 새 CSV 단건 조사용.
- `diff`: 두 빌드의 타입 diff 단독 실행.

**`verify` 서브커맨드는 만들지 않음** (§ 6.4 근거). 사람 문서와 코드의 일치를 강제하는 것은 원칙 2에 어긋나며 자기참조 루프.

### 7.3 클리셰 방지

"`.csv` 여러 개를 입력으로 받는 `gleaner batch` 같은 범용 배치"는 만들지 않는다. **Noumenon의 엔티티 3개(+보조)는 고정 스코프**이므로 `extract`는 내부적으로 Item/Recipe/ClassJob 파이프라인을 하드코딩한다. 나중에 Action/Quest가 추가되면 그때 파이프라인을 추가.

---

## 8. 모듈/파일 구조 제안

```
packages/gleaner/
├── Cargo.toml
├── overrides/
│   └── optional.toml                   # 데이터 기반 판정 강제 오버라이드
├── src/
│   ├── main.rs                         # 계승 (CLI dispatch 축소)
│   ├── cli.rs                          # 교체 (서브커맨드 구조)
│   ├── constants.rs                    # 계승 (오버라이드 테이블 추가 안 함)
│   │
│   ├── parse/                          # Layer 1 (신규, 기존 builder.rs 분할)
│   │   ├── mod.rs
│   │   ├── csv_header.rs               # 3행 헤더 파싱
│   │   ├── type_inference.rs           # FieldType 결정
│   │   ├── normalize.rs                # 결정론적 필드명 변환 (오버라이드 없음)
│   │   ├── row.rs                      # 행 데이터 → RawValue 변환
│   │   ├── exd_header.rs               # exd-header.json 인덱싱
│   │   └── custom_type_graph.rs        # 커스텀 타입 재귀 탐색
│   │
│   ├── schema/                         # 기존 모듈 유지
│   │   ├── mod.rs
│   │   ├── types.rs                    # 계승 + FieldType variant 추가
│   │   ├── error.rs                    # 계승 + variant 추가
│   │   └── typescript.rs               # 계승 (도메인 객체 출력으로 전환)
│   │
│   ├── enrich/                         # Layer 2 (신규, 수동 어댑터 없음)
│   │   ├── mod.rs
│   │   ├── normalize.rs                # 익명 드롭, 일반 정규화 보강
│   │   ├── array.rs                    # 고정 배열 collapse
│   │   ├── base_param.rs               # BaseParam 이름 embed
│   │   ├── recipe_level.rs             # RecipeLevelTable flatten
│   │   ├── optional.rs                 # 데이터 기반 판정 + override 로드
│   │   ├── item.rs                     # Item 도메인 조립 (raw 패스 스루)
│   │   ├── recipe.rs                   # Recipe 도메인 조립
│   │   └── class_job.rs                # ClassJob 도메인 조립
│   │
│   ├── domain/                         # 도메인 모델 (신규)
│   │   ├── mod.rs
│   │   ├── item.rs
│   │   ├── recipe.rs
│   │   ├── class_job.rs
│   │   └── base_param.rs
│   │
│   ├── serialize/                      # Layer 3 (신규)
│   │   ├── mod.rs
│   │   ├── item.rs
│   │   ├── recipe.rs
│   │   ├── class_job.rs
│   │   ├── base_param.rs
│   │   ├── item_ui_category.rs
│   │   ├── typescript.rs               # .d.ts 출력
│   │   ├── diff.rs                     # 이전 빌드와의 .d.ts diff
│   │   └── writer.rs                   # 공용 결정적 쓰기 헬퍼
│   │
│   └── utils/                          # 계승
│       └── analyze.rs                  # analyze_missing_files
│
└── tests/
    ├── fixtures/                       # 샘플 CSV (키 3~5개 규모)
    │   ├── Recipe.csv
    │   ├── ClassJob.csv
    │   └── ...
    └── golden/                         # 골든 JSON (스냅샷 테스트)
        ├── recipe/*.json
        └── classJobs.json
```

---

## 9. 구현 순서 (마일스톤)

### M1. 리팩토링: 기존 builder.rs 분할 (레이어 1 준비)
- `parse/csv_header.rs`, `parse/type_inference.rs`, `parse/custom_type_graph.rs`로 쪼갬.
- 동작 동치성 확인: 현 CLI로 동일한 `.d.ts`가 나오는지 회귀 테스트.
- **외부 동작 변화 없음**. 이 단계에서 커밋 안정화.

### M2. Layer 1 완성: 행 데이터 변환 + 결정론적 정규화
- `parse/row.rs`: `StringRecord` → `RawValue` 변환.
- `parse/normalize.rs`: 결정론적 필드명 알고리즘 (오버라이드 테이블 없음).
- `CsvData { schema, records }` 조립.
- 단위 테스트: ClassJob.csv 샘플 3행 파싱 → `RawRecord` 검증.
- 의존성 추가: `serde`, `serde_json`.

### M3. Layer 2 스켈레톤: 패스 스루 + 익명 드롭
- `enrich/item.rs`: raw 전 필드를 도메인 객체로 패스 스루 (수동 어댑터 없음).
- 익명 필드만 제외.
- Item 한 행 → `domain::Item` 생성 성공까지.

### M4. Layer 2 확장: FK 해석 + flatten + optional 판정
- `enrich/base_param.rs`, `enrich/recipe_level.rs`.
- `enrich/optional.rs`: 데이터 기반 판정 + `overrides/optional.toml` 로드.
- Recipe 한 행 → `domain::Recipe` with `recipeLevel`/`maxQuality` 검증.
- `reports/optional-analysis.md` 1차 출력.

### M5. Layer 3 스켈레톤: 단일 엔티티 직렬화
- `serialize/class_job.rs`부터 (가장 작은 엔티티).
- `classJobs.json` 출력 후 [[data-contract]]의 ClassJob 형태와 사람이 한 번 비교.

### M6. 전체 파이프라인: `gleaner extract`
- `cli.rs` 서브커맨드 구조 구현.
- Item/Recipe/ClassJob/BaseParam 모두 출력.
- 출력 크기/개수 예상치(2MB/38k/200KB/4k/5KB/1KB) 검증.

### M7. 타입 출력 + diff 리포트
- `.d.ts` 생성 (도메인 객체 기준).
- `reports/schema-diff.md` (이전 빌드와 비교).
- CI에서 `--prev-snapshot ./dist-main`으로 매 PR마다 diff 출력.

### M8. 성능/결정성 마무리
- `rayon` 도입 여부 결정 (38k 개별 파일 쓰기 실측).
- 출력 결정성 테스트 (2회 실행 → diff 없음).

---

## 10. 리스크 & 판단 보류

### 10.1 `exd-header.json` 해석 포맷 미확정
7,777 테이블 스키마가 어떤 구조인지는 실제 파싱 시점에 확인. 포맷이 단순하면 Layer 1에서 바로 소비, 복잡하면 별도 `parse/exd_header.rs`를 두고 캐시.

### 10.2 optional 판정 임계치
- 30% null/zero를 기본 임계치로 시작.
- raw 데이터 분포가 고르지 않을 수 있으니 M4 이후 실측해 조정.
- 오버라이드 파일이 비대해지면 알고리즘이 부족하다는 신호.

### 10.3 `itemSummaries.json` 압축 포맷
- 객체 배열이 기본. 필요 시 Gleaner가 숫자 배열 압축 옵션(`--compact-summaries` 플래그)을 제공.
- 판단: **초기엔 객체 배열만**, 실제 2MB 성능 이슈가 확인되면 압축 도입.

### 10.4 raw 변화 빈도와 diff 리포트 위치
- 게임 패치 주기에 raw가 갱신되므로 diff는 빈번하지는 않으나 발생 시 임팩트가 큼.
- diff 리포트를 어디에 두고 누가 읽을지: 코드 저장소 `dist/reports/`에 두되, 의미 있는 변경이 있으면 PR 본문에도 첨부하는 자동화 고려.

---

## 11. 다음 단계

- [[frontend-plan]] (Phase C): Gleaner가 생성한 `.d.ts`를 import하는 Frontend 설계. 표시 라벨 맵의 위치, 엔티티 그래프 탐색 UX, 기술 스택.
- Phase D `roadmap.md`: M1~M8 마일스톤과 Frontend 작업을 실행 순서로 엮는다.
