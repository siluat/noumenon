# Known Quirks (Phase D-0 산출물)

#unreviewed

기존 `web-projects/packages/noumenon-gleaner/src/`(약 1,514줄, 9개 파일)의 코드를 *spec과 별개로* 읽고 추출한 **암묵 지식 메모**. 신규 레포(`siluat/noumenon`)에서 fresh re-implement할 때 *동일한 함정에 다시 빠지지 않기 위한* 체크리스트.

- 상위: [[noumenon]]
- 선행: [[roadmap]] (D-0 단계의 산출물)
- 분석 대상: `/Users/siluat/Projects/web-projects/packages/noumenon-gleaner/src/`
- 분석 일시: 2026-04-19

> 💡 **인사이트 — 이 문서의 목적**
> 
> Phase C의 [[gleaner-redesign]]은 *"무엇을 만드는가"*를 정의합니다. 하지만 spec에는 *"raw 데이터의 어떤 기벽 때문에 어떤 분기가 필요한지"*가 거의 안 나옵니다 — 그건 *원본을 만지면서 알게 되는 지식*이기 때문입니다. 이 문서는 그런 *발견된 사실들*을 모은 것이며, 새 구현이 같은 발견 과정을 다시 겪지 않게 합니다.

---

## 1. 헤더 파싱의 비명시 규칙

### 1.1 헤더 행은 *순서가 아니라 식별자*로 찾는다

기존 코드(`builder.rs:241~315`)는 3행 헤더를 *위치(0/1/2번째 행)* 가 아니라 *내용*으로 찾습니다.

| 헤더 종류 | 식별자 | 코드 위치 |
|---|---|---|
| field_names | 첫 컬럼이 정확히 `"key"` | `find_field_names_row` |
| field_descriptions | 첫 컬럼이 정확히 `"#"` | `find_field_descriptions_row` |
| field_types | 첫 컬럼이 `BASIC_TYPES`에 속함 | `find_field_types_row` |

**이유**: SaintCoinach가 추출하는 raw CSV의 헤더 순서가 *항상 동일하지 않을 수 있다*는 가정. 실제 데이터에서 마주친 적이 있는지는 확인 안 됐지만, *방어적 코드*로 들어가 있음.

**신규 구현 권고**: 동일한 패턴 유지. 추가로 *헤더가 0개 발견될 때*와 *2개 이상 발견될 때* 모두 명시적 에러(`MissingCsvHeader` / `DuplicateCsvHeader`)로 구분해야 디버깅이 쉬움.

### 1.2 BOM(Byte Order Mark) 제거

`builder.rs:247` — `first_col.trim_start_matches('\u{feff}').trim()`

UTF-8 BOM(`\u{feff}`)이 첫 컬럼 앞에 붙어있을 수 있음. 이걸 제거하지 않으면 `"key"` 식별이 실패하고 `MissingCsvHeader` 에러가 남.

> 💡 **인사이트 — BOM은 *발견된 후에야* 알게 되는 종류의 기벽**
> 
> Excel이나 일부 SaintCoinach 변종이 UTF-8 출력 시 BOM을 붙이는 경우가 있습니다. 코드에서 1줄로 처리되지만, *처음 마주쳤을 때* 원인 파악에 시간이 듭니다 (보통 *"왜 'key'를 못 찾지?"* 한참 헤맴). 신규 구현에서 *1행 1컬럼*은 항상 `trim_start_matches('\u{feff}').trim()`을 거쳐야 합니다.

### 1.3 최소 3행 제약

`builder.rs:106~111` — records가 3행 미만이면 `InvalidFormat`. 빈 CSV나 헤더만 있는 CSV의 명시적 거부.

---

## 2. 필드명 정규화 (`sanitize_field_name`) 상세

`builder.rs:328~388`의 알고리즘이 [[feature-scope]] § 5의 핵심.

### 2.1 변환 규칙 (실제 코드 기준)

```
입력 description 한 글자씩 순회:
  영문자/언더스코어 → result에 추가
  숫자 (단, 첫 글자가 아닐 때) → result에 추가
  '{' '}' ' ' '-' '.' → separator로 취급, 다음 영문자 대문자화 플래그
  그 외 → 무시 (drop)

후처리:
  빈 문자열 → "_"
  숫자로 시작 → "_" prefix
  to_camel_case 적용
```

### 2.2 실제 변환 사례

| Raw description | 변환 결과 | 비고 |
|---|---|---|
| `Item{Result}` | `itemResult` | `{` `}`가 separator |
| `Item{Ingredient}[0]` | `itemIngredient0` | `[`도 무시되며 `0`이 붙음 |
| `Name{English}` | `nameEnglish` | 동 |
| `IsSecondary` | `isSecondary` | 첫 글자만 소문자 |
| `Modifier{HitPoints}` | `modifierHitPoints` | |
| `# 3` (익명) | `3` → `_3` (숫자 prefix) | 숫자만 남으면 `_` prefix |

### 2.3 D-2 신규 구현 시 변경점

[[gleaner-redesign]] § 4.2의 `FixedArrayElement` variant는 기존 코드에 *없습니다*. 기존은 `Item{Ingredient}[0]`이 `itemIngredient0` 식으로 *한 컬럼 단위*로 처리됩니다. **신규 구현은 Layer 1에서 `FixedArrayElement { base, scope, index }` 변종으로 인식하고, Layer 2에서 collapse**하는 것이 spec.

### 2.4 중복 필드명 자동 suffix

`builder.rs:167~172` — 같은 이름이 두 번 나오면 `name`, `name1`, `name2`로 자동 번호. 이는 *오케이지만, silent suffix가 디버깅을 어렵게 할 수 있음*. 신규 구현에서는 **로그 출력**(어떤 description이 어떤 suffix를 받았는지)을 추가 권장.

---

## 3. Type 추론의 corner cases

### 3.1 description `"Key"`는 type 행과 무관하게 `FieldType::Key`로 강제

`builder.rs:137~141`:

```rust
let field_type = if description == "Key" {
    FieldType::Key
} else {
    self.parse_field_type(type_str, base_dir, csv_path)?
};
```

CSV의 type 행이 다른 값이어도, description이 정확히 `"Key"` 문자열이면 무조건 `FieldType::Key`. 일종의 **description-driven type override**.

> 💡 **인사이트 — 왜 이 분기가 필요한가**
> 
> raw CSV의 type 행이 *항상 신뢰할 만하지 않다*는 사실의 단편적 증거. spec § 4.4의 *"`ClassJob.Name{English}`의 byte 표기는 거짓"*과 같은 계열 — *type 행에 적힌 게 진실이 아닐 수 있다*. 신규 구현에서는 이 임시방편 대신 [[gleaner-redesign]] § 4.4의 `exd-header.json` 교차 검증으로 일반화하는 게 spec.

### 3.2 description `"#"`로 시작 → field name `"id"` 강제

`builder.rs:144~145` — 첫 번째 필드의 description이 `#`로 시작하면 *무조건* `"id"`. raw CSV의 첫 컬럼이 `key`(integer)인 패턴 처리.

### 3.3 description 비어있을 때 fallback 체인

`builder.rs:147~158`:

1. name이 `"key"`면 → `"id"`
2. name이 숫자로만 구성됐으면 → `field{N}` (예: `0` → `field0`)
3. 그 외 → name 그대로 사용

raw CSV에서 *익명 컬럼*(헤더가 비어있는 컬럼)의 처리. spec § 4.2의 `Anonymous` variant와 대응.

### 3.4 Unknown type → String fallback (위험)

`builder.rs:228~230`:

```rust
// Unknown types default to string
else {
    Ok(FieldType::String)
}
```

알 수 없는 type 문자열은 *조용히* `String`으로 처리. **silent failure**의 전형. 신규 구현에서는:

- (권고) `FieldType::Unknown(String)` variant 추가, *경고 로그* 출력
- 또는 명시적 에러 (다만 raw 변화 시 빌드 전체 실패하므로 trade-off)

### 3.5 Bit type은 16진수

`utils.rs:46` — `u8::from_str_radix(s, 16)` — `bit&FF` = 255. `bit&01` = 1. 진수 파싱이 16진수임을 모르면 잘못된 값이 나옴.

### 3.6 PascalCase fallback for custom type

`utils.rs:34` — `trimmed.chars().next().is_some_and(|c| c.is_uppercase())` — type 문자열이 대문자로 시작하면 *그것만으로* custom type으로 간주.

이는 `CUSTOM_TYPE_PATTERNS` (constants.rs의 7개 단어 휴리스틱)와 결합돼 작동.

---

## 4. **`CUSTOM_TYPE_PATTERNS`: Phase C 원칙과 충돌하는 휴리스틱**

### 4.1 현 코드의 모습

`constants.rs:7~9`:

```rust
pub const CUSTOM_TYPE_PATTERNS: &[&str] = &[
    "Category", "Action", "Level", "Param", "Job", "Company", "Series",
];
```

type 문자열에 이 7개 단어 중 하나가 *포함*되면 custom type으로 간주.

### 4.2 충돌 지점

[[gleaner-redesign]] § 0 원칙 3:

> **결정론적 알고리즘만으로 변환**. 사람이 손으로 관리하는 "필드명 화이트리스트/오버라이드 테이블"은 데이터 계약의 일부가 될 수 없음.

`CUSTOM_TYPE_PATTERNS`는 *사람이 관리하는 하드코딩 단어 목록*입니다. 새 raw 엔티티가 추가되며 이 목록에 없는 단어를 쓰면 (예: `Quest`, `Status`, `Town`...) 기존 코드는 *PascalCase fallback*에 의지해 어찌저찌 인식하지만, *명시적 가이드*는 사라집니다.

### 4.3 신규 구현의 대체안

세 가지 후보:

| 안 | 방법 | 장단 |
|---|---|---|
| **A** | "PascalCase로 시작 + 같은 dir에 `{Type}.csv` 존재"로 판별 | 결정론적, file system 기반. 추가 설정 불필요 |
| **B** | `exd-header.json`을 진실로 사용 (해당 column이 reference type인지 확인) | spec § 4.4와 일관. exd-header.json 의존 |
| **C** | A + B 결합 (exd-header가 우선, 없으면 file 존재 검사) | 가장 견고, 약간 복잡 |

**권고**: **C** — exd-header.json이 권위 있는 reference 정보를 담고 있다면 그걸 우선 신뢰하고, 누락된 항목은 file 존재 검사로 fallback. 어느 경우에도 `CUSTOM_TYPE_PATTERNS` 같은 단어 목록은 부활시키지 않음.

> 💡 **인사이트 — 왜 단어 목록이 자라기 시작하면 멈출 수 없는가**
> 
> "어, `Quest`도 추가해야겠네"가 한 번 들어오면, 다음에 `Town`, `Aetheryte`, `MountFamily`... 끝없이 자랍니다. 그리고 *언제 추가해야 하는지의 기준*이 없으므로, 누락이 발생해도 빌드는 통과합니다(PascalCase fallback 때문). 결국 *목록은 자라지만 정확성은 보장되지 않는* 최악의 상태가 됩니다. 이게 원칙 3이 *완전 금지*를 명시하는 이유입니다.

---

## 5. 순환/자기 참조 처리 (실제 동작)

### 5.1 순환 참조는 *감지 후 허용*

`builder.rs:62~67`:

```rust
if self.processing_stack.contains(&schema_name.to_string()) {
    // If we're already processing this schema, just return the name
    // This allows circular references to be resolved later
    return Ok(schema_name.to_string());
}
```

처리 중인 스키마가 다시 참조되면 *에러 없이* 이름만 반환. 결과적으로 양방향 FK가 가능. 자기 참조(`ClassJob.classJobParent` → `ClassJob`)도 같은 메커니즘으로 동작.

기존 코드의 테스트(`test_circular_dependency_allowed`, `test_self_reference_allowed`)가 이 동작을 *명시적으로 보장*함.

### 5.2 신규 구현 시 유지

[[data-contract]] § 3.3에 *"자기참조 (Item ↔ Item) → ID 참조 (영구)"*가 명시. 코드 동작과 일치.

---

## 6. 에러 진단 메시지 (사용자 친화 자산)

`main.rs:55~94`의 에러 분기별 친절한 메시지는 **반드시 계승**할 자산입니다.

### 6.1 분기

- `FileNotFound` → 누락 파일 자동 분석 (`analyze_missing_files`) + "Suggested files to create" 출력
- `InvalidFormat` → CSV 형식 가이드 출력 (예시 포함)
- 그 외 → 일반 에러 메시지

### 6.2 raw 데이터 작업 시의 가치

게임 패치 후 raw가 변하면 *어떤 CSV가 새로 생기거나 사라졌는지*를 즉시 알려줍니다. 1인 프로젝트에서 디버깅 시간을 결정적으로 줄여주는 기능. 신규 구현에서 **CLI 레이어에 그대로 이식** 권장.

> 💡 **인사이트 — 친절한 에러는 "성능 외 최고의 미적분"**
> 
> 1인 프로젝트에서 *코드를 다시 읽는 사람*은 6개월 후의 자기 자신입니다. 그 사람을 위해 친절한 에러 메시지를 남겨두는 건 *미래의 자신에 대한 최고의 투자*입니다. 기존 코드의 이 부분만큼은 *spec과 무관하게* 그대로 가져갈 가치가 있습니다.

---

## 7. 캐싱과 결정성

### 7.1 스키마 캐싱

`builder.rs:58~60` — 같은 schema가 두 번 요청되면 즉시 return. 동일 CSV가 여러 곳에서 참조돼도 1회만 파싱. Layer 1/2/3 분리 후에도 동일한 캐싱 보장 필요.

### 7.2 정렬을 통한 결정적 출력

`typescript.rs:45~46` — 출력 전 schema들을 이름순 정렬:

```rust
let mut sorted_schemas: Vec<&Schema> = schemas.values().collect();
sorted_schemas.sort_by(|a, b| a.name.cmp(&b.name));
```

`HashMap` 순회의 비결정성을 배제하기 위한 정렬. [[gleaner-redesign]] § 6.2의 결정성 요건과 일치. 신규 구현에서는 `IndexMap` 또는 동일 정렬 패턴 유지.

---

## 8. 기존 코드에 *없는* 것 (재실측)

[[gleaner-redesign]] § 1.3의 *"현 구현에 없는 것"* 7개 항목 검증:

| 항목 | 실제 부재 확인 | 비고 |
|---|---|---|
| 행 데이터 변환 (CSV row → JSON) | ✅ 부재 | `RawValue` 타입 자체가 없음. records는 메타만 |
| FF14 메타 문법 (`X{Y}[N]`) 정규화 | △ 부분적 | `sanitize_field_name`이 단일 컬럼은 처리. 배열 collapse는 없음 |
| 도메인 보강 (RecipeLevelTable flatten 등) | ✅ 부재 | enrich 레이어 자체 없음 |
| JSON 직렬화 (`serde`) | ✅ 부재 | `Cargo.toml`에 serde 의존 없음 |
| 다수 CSV → 다수 JSON 파이프라인 | ✅ 부재 | CLI는 단일 입출력 |
| 이전 빌드 대비 schema diff | ✅ 부재 | diff 로직/출력 없음 |
| 데이터 기반 optional 판정 | ✅ 부재 | 행 데이터를 안 읽으므로 분포 통계 불가 |

**결론**: spec의 부재 분석은 정확. D-2~D-5는 *완전 신규 구현* 범위.

---

## 9. D-2/D-3 구현 시 체크리스트 (총정리)

신규 구현이 같은 함정에 빠지지 않게 하는 작업 진입 시 점검 항목.

### 9.1 Layer 1 진입 시

- [ ] CSV 1행 1컬럼은 `trim_start_matches('\u{feff}').trim()` 적용 (BOM)
- [ ] 헤더 3행은 *위치가 아닌 식별자*로 검색
- [ ] 헤더 행이 0개/2개 이상일 때 명시적 에러 분기
- [ ] `description == "Key"` 분기는 *임시*임 — `exd-header.json`으로 일반화 목표
- [ ] `description.starts_with('#')` → field name `"id"` 패턴 유지
- [ ] 익명 컬럼 (description 비어있음 + name이 숫자) 인식 후 `Anonymous` variant
- [ ] 중복 필드명 발생 시 *suffix 자동 부여 + 로그 출력*
- [ ] Unknown type은 `Unknown(String)` variant + 경고 (silent String fallback 금지)
- [ ] `bit&XX`는 *16진수* 파싱

### 9.2 Layer 2 진입 시

- [ ] `CUSTOM_TYPE_PATTERNS` 같은 단어 목록 *부활 금지* (원칙 3)
- [ ] custom type 판별은 (a) `exd-header.json` reference 정보, (b) file 존재 검사 순
- [ ] 순환 참조는 *감지 후 허용* (양방향 FK / 자기 참조 정상 케이스)
- [ ] 동일 schema 캐싱 유지 (1회만 파싱)
- [ ] 익명 필드 드롭 시 `exd-header.json`로 이름 복구 시도

### 9.3 Layer 3 진입 시

- [ ] `serialize_json` 전 entity 배열은 *항상 id 오름차순* 정렬
- [ ] HashMap 순회 비결정성 배제 (`IndexMap` 또는 정렬)
- [ ] Schema 출력도 이름 순 정렬 (기존 typescript.rs 패턴 유지)

### 9.4 CLI/UX 진입 시

- [ ] `FileNotFound` → `analyze_missing_files` 자동 호출 + "Suggested files to create" 출력
- [ ] `InvalidFormat` → CSV 예시 포함 가이드 출력
- [ ] 모든 에러 메시지에 *어디서/왜* 정보 포함 (path + reason 동시 노출)

---

## 10. 다음 단계

D-0 종료. 신규 레포에서 D-2~D-3 구현 시 §9 체크리스트를 코드 옆에 두고 진행.

이 문서는 D-1 시점에 신규 레포 `docs/known-quirks.md`로 이관된다. 이관 후에도 dotori의 마스터 사본은 작성 이력 보존용으로 유지하되, 갱신은 신규 레포 쪽만 반영한다 (다른 Phase C 문서들과 동일 정책).

> 💡 **인사이트 — 이 문서가 시간이 지나며 어떻게 변할까**
> 
> 가장 이상적인 시나리오: D-2~D-3을 진행하며 *spec과 코드와 이 문서*를 함께 보면서, *체크리스트 항목 하나씩 줄을 그어가며* 작업합니다. D-3가 끝날 때쯤 §9의 체크리스트는 대부분 *해결됨*으로 표시되고, 그때 새로 발견된 기벽이 §11 같은 새 절로 추가됩니다. 6개월 후 다시 보면 *"아 그때 이런 함정이 있었지"*를 회상하는 *project lore* 문서가 됩니다.
