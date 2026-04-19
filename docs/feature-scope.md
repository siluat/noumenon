# Feature Scope (Phase B)

#unreviewed

Noumenon의 "**어떤 필드를 UI에 노출할지**"를 엔티티/필드 단위로 확정한 문서. [[data-contract]]와 [[frontend-plan]]이 이 분류를 참조한다.

- 상위: [[noumenon]]

---

## 중요 원칙

### 원칙 1. raw-data가 스키마의 단일 진실 (single source of truth)

`ff14-raw-data`의 CSV 파일들이 모든 엔티티 스키마의 **유일한 진실의 출처**다. raw-data는 외부(게임 클라이언트)에서 오며, 자주 변하지 않지만 **언제든 변경될 수 있다**. 따라서:

- 서비스 측에 "영원히 고정된 스키마"가 있다는 전제를 두면 안 됨.
- 사람이 손으로 관리하는 "필드 화이트리스트"가 진실을 결정해서는 안 됨.
- raw의 변화는 자동으로 서비스까지 흘러야 하며, 흐름 중간의 수동 단계는 **silent divergence 지점**.

### 원칙 2. 추출 계약 = raw 패스 스루 (자동)

Gleaner는 raw의 **익명 아닌 전 필드**를 결정론적 알고리즘으로 JSON에 옮긴다. 사람이 "이 필드는 드롭"을 결정해 추출 계약에서 제외하지 않는다.

- raw에 새 필드가 추가되면 → 자동으로 JSON에 포함됨.
- raw에서 필드명이 바뀌면 → Gleaner의 **변화 리포트**로 표면화 (사람이 인지).
- raw에서 필드가 사라지면 → 타입 diff로 즉시 표면화.

### 원칙 3. UI 노출 계약 ≠ 추출 계약

이 문서의 **MUST / NICE / LOW 분류**는 UI 노출 계약에 대한 것이다. 추출은 자동이고, UI 노출은 의도적 선별이다.

- **MUST**: Detail 페이지 기본 탭에 즉시 노출.
- **NICE**: Detail 확장 탭/토글/접근성 낮은 섹션에 노출 (또는 M2 이후 추가).
- **LOW**: 현재 UI에 노출 계획 없음. **단, JSON에는 그대로 포함됨**. 나중에 필요해지면 Frontend 코드만 수정하면 됨 (Gleaner 재실행/CDN 재업로드 불필요).

기존 "OUT(드롭)" 용어는 이 문서에서 **폐기**한다. raw-single-truth 원칙과 충돌하기 때문.

### 원칙 4. 익명 필드만 실제 드롭

raw CSV의 헤더가 비어있는("# 3", "# 44" 식 인덱스만 남은) 컬럼은 **현재 해석 수단 없음**이므로 Gleaner 출력에서 제외. 단:

- `exd-header.json`에 해당 인덱스의 이름이 복구돼 있다면 되살림.
- 익명 드롭은 "영구 결정"이 아닌 "**해석 불가 상태의 기본값**"임.

---

## 1. Item

**추출**: raw `Item.csv`의 익명 아닌 전 필드 자동. (패스 스루)

**UI 노출 기준선**: 사용자 가치 기반으로 약 58개 필드를 MUST + NICE의 상한선으로 삼는다. raw에 그 외 필드가 있다면 LOW로 두고, Frontend 리뷰에서 승격 고려.

### MUST/NICE 분류 (UI 노출 기준)

MUST (Item Detail 기본 노출):
- 기본 메타: id, name, description, icon, itemLevel, rarity, itemUICategory, stackSize
- 거래/보관 플래그: isUnique, isUntradable, isIndisposable, midPrice, lowPrice
- 장비 품질: canBeHq, isDyeable
- 장비 연계 FK: repairClassJob, repairItem, glamourItem, useClassJob
- 장착 제약: equipLevel, equipRestriction, classJobCategory
- 전투 능력치: physDamage, magDamage, delay, block, blockRate, physDefense, magDefense, baseParam0~5
- 메테리아: materiaSlotCount, isAdvancedMeldingPermitted
- 수집 플래그: isCollectable, alwaysCollectable

NICE (확장/토글):
- lot, isCrestWorthy, cooldown, salvage, aetherialReduce
- grandCompany, itemSeries, baseParamModifier
- itemSpecialBonus, itemSpecialBonusParam, specialBaseParam0~5
- materializeType, isPvP, isGlamourous

LOW: 현재 없음 (58필드가 모두 노출 후보). 향후 raw에 추가된 필드는 기본 LOW에서 시작.

### 근거

- 사용자 가치 검증을 통과한 약 58필드를 MUST + NICE의 상한선으로 채택.
- 추출은 어차피 자동이므로 MUST/NICE 구분은 "화면 레이아웃 결정"의 문제로 좁혀짐.

---

## 2. Recipe

**추출**: raw `Recipe.csv`의 익명 아닌 전 필드 자동. (패스 스루)

**RecipeLevelTable flatten**: Gleaner가 조인해 `recipeLevel`, `maxQuality`를 Recipe 레코드에 파생 필드로 포함. Frontend는 RecipeLevelTable의 존재를 모름.

**재료 배열 재구성**: raw는 `Item{Ingredient}[0..7]` 고정 8슬롯. Gleaner가 id 0 슬롯을 양쪽(id와 amount)에서 동일 인덱스로 제거해 가변 배열(`itemIngredients`, `amountIngredients`)로 출력. id 인덱스 동기화 보장.

### UI 노출 분류

MUST (Recipe Detail 기본 노출):

| 서비스 필드명 | Raw CSV 헤더 | 비고 |
|---|---|---|
| id | key | |
| craftType | CraftType | ClassJob FK |
| itemResult | Item{Result} | Item FK |
| amountResult | Amount{Result} | |
| itemIngredients | Item{Ingredient}[0..7] | flatten 후 배열 |
| amountIngredients | Amount{Ingredient}[0..7] | |
| requiredCraftsmanship | RequiredCraftsmanship | |
| requiredControl | RequiredControl | |
| difficultyFactor | DifficultyFactor | |
| qualityFactor | QualityFactor | |
| durabilityFactor | DurabilityFactor | |
| canHq | CanHq | |
| isSecondary | IsSecondary | |
| recipeLevel | (RecipeLevelTable flatten) | Gleaner 파생 |
| maxQuality | (RecipeLevelTable flatten) | Gleaner 파생 |

NICE (Recipe Detail 확장, M2 이후 UI):
- canQuickSynth, quickSynthCraftsmanship, quickSynthControl
- itemRequired (도구/재료 외 필요 아이템)
- questUnlock, secretRecipeBook
- materialQualityFactor, requiredQuality

LOW (JSON 포함, 현재 UI 계획 없음):
- expRewarded, isSpecializationRequired, isExpert, patchNumber, displayPriority, status{Required}
- raw의 기타 파생/인덱스 컬럼 (해석된 이름이 있는 한 전부 JSON에 포함)

---

## 3. ClassJob

**추출**: raw `ClassJob.csv`의 익명 아닌 전 필드 자동. (패스 스루)

### UI 노출 분류

MUST (ClassJob Detail 기본 노출):

| 서비스 필드명 | Raw CSV 헤더 | 비고 |
|---|---|---|
| id | key | |
| name | Name | "검술사", "나이트" |
| abbreviation | Abbreviation | "GLA", "PLD" |
| nameEnglish | Name{English} | "Gladiator" (타입 교정 필요) |
| jobIndex | JobIndex | 0=class, 1+=job |
| classJobCategory | ClassJobCategory | |
| classJobParent | ClassJob{Parent} | 자기참조 FK |
| role | Role | 탱/힐/딜/제작/채집 |
| primaryStat | PrimaryStat | |
| itemStartingWeapon | Item{StartingWeapon} | Item FK |
| itemSoulCrystal | Item{SoulCrystal} | Item FK |
| uiPriority | UIPriority | |
| isLimitedJob | IsLimitedJob | |
| canQueueForDuty | CanQueueForDuty | |

NICE (ClassJob Detail 확장):
- 능력치 가중치: modifierHp, modifierMp, modifierStr, modifierVit, modifierDex, modifierInt, modifierMind, modifierPiety
- 한계 기술: limitBreak1, limitBreak2, limitBreak3 (각 Action FK)
- 기타: startingLevel, startingTown

LOW:
- expArrayIndex, battleClassIndex, dohDolJobIndex, pvpActionSortRow, unlockQuest, relicQuest, prerequisite, monsterNote, partyBonus
- raw의 기타 해석된 필드 전부 (JSON에 포함)

---

## 4. 엔티티 간 관계 지도

```
           ┌─────────────────────────────────┐
           │                                 │
  Item ◄───┤ Recipe.itemResult               │
  Item ◄───┤ Recipe.itemIngredients[]        │
  Item ◄───┤ Recipe.itemRequired (NICE)      │
  Item ◄───┤ ClassJob.itemStartingWeapon     │
  Item ◄───┤ ClassJob.itemSoulCrystal        │
  Item ◄───┤ Item.repairItem                 │
  Item ◄───┤ Item.glamourItem                │
           │                                 │
ClassJob ◄─┤ Item.repairClassJob             │
ClassJob ◄─┤ Item.useClassJob                │
ClassJob ◄─┤ Recipe.craftType                │
ClassJob ◄─┤ ClassJob.classJobParent (자기참조)│
           │                                 │
           └─────────────────────────────────┘
```

### 외부 미포함 엔티티 처리 방침

raw에 별도 테이블이 존재하지만 이번 스코프에 포함되지 않는 참조 대상:

| 참조 대상 | 처리 방침 |
|---|---|
| ClassJobCategory | raw ID 값 보존. 19직업 포함 플래그 해석은 향후 확장 |
| ItemUICategory | 소규모 참조 테이블을 summaries와 함께 embed |
| ItemSeries, GrandCompany | raw ID 값만 보존 |
| BaseParam | 이름 테이블 별도 추출 (UI의 `baseParam0~5` 해석용, 소규모) |
| RecipeLevelTable | Recipe에 flatten (별도 테이블 없음) |
| Quest, Status, Action, SecretRecipeBook | NICE 필드 참조. ID만 보존, 추출 대상 아님 (이 스코프 한정) |

---

## 5. Gleaner 필드명 정규화 규칙

FF14 raw CSV의 메타 문법 → 서비스 필드명 변환. **결정론적 알고리즘만** 사용 (오버라이드 테이블 없음).

| Raw 패턴 | 변환 규칙 | 예시 |
|---|---|---|
| `X{Y}` | `xY` (camelCase 결합) | `Item{Result}` → `itemResult` |
| `X{Y}[N]` | 동일 base/scope의 인덱스 컬럼을 배열로 collapse | `Item{Ingredient}[0..7]` → `itemIngredients[]` |
| `X[N]` | 배열로 collapse (또는 개별 슬롯 유지 — § 5.1 참조) | `BaseParam[0..5]` → 슬롯별 |
| `Modifier{X}` | `modifierX` (축약 없음) | `Modifier{HitPoints}` → `modifierHitPoints` |
| `IsX` / `CanX` | 그대로 camelCase | `IsSecondary` → `isSecondary` |
| 나머지 PascalCase | camelCase 변환 | `StackSize` → `stackSize` |

### 5.1 예외 정책

- **축약 이름 예외 없음**: `physDamage`(←`PhysicalDamage`) 같은 사람 손맛 축약은 채택하지 않고 알고리즘 결과(`physicalDamage`)를 사용한다. UI 라벨에서 축약이 필요하면 Frontend의 **표시 라벨 맵**에서 해결 (데이터 스키마 오염 금지).
- **`BaseParam[0..5]` 슬롯**: 배열 대신 `baseParam0..5`로 슬롯별 유지 (UI가 "첫번째 능력치 슬롯"을 참조하는 패턴이 있음). 이것도 알고리즘으로 처리 가능하지만, 슬롯 의미가 있으므로 개별 필드 유지.
- **익명 필드**: 헤더가 비어있는 컬럼은 드롭 (원칙 4). `exd-header.json` 교차 참조로 복구 시도.
- **`ClassJob.Name{English}` 타입 교정**: CSV 타입 행은 `byte`이지만 실제 데이터가 문자열. Gleaner가 `exd-header.json`과 교차 검증해 String으로 강제 (raw CSV의 타입 표기가 진실이 아닐 때 대비).

### 5.2 raw 변화 리포트 (원칙 1, 2의 실체화)

Gleaner가 추출 실행 시마다 **이전 빌드와의 스키마 diff**를 출력:

- 새 필드 (+): 자동으로 JSON에 포함됨. UI 노출은 이 문서에서 선택.
- 사라진 필드 (−): 타입 diff로 즉시 표면화. 소비자(Frontend, data-contract 문서) 업데이트 필요.
- 이름 변경: 휴리스틱(인덱스 위치 + 타입 일치)으로 감지. 확정은 사람이.

이 리포트가 "raw 변화를 silent divergence 없이 인지"하는 장치.

---

## 6. Phase B 확정 사항 요약

| 항목 | 결정 |
|---|---|
| 추출 계약 | raw 익명 아닌 전 필드 자동 패스 스루 |
| UI 노출 계약 | 이 문서의 MUST/NICE 분류 (LOW는 JSON 포함, UI 미노출) |
| Item | 전 필드 추출 + 약 58필드를 MUST/NICE로 분포 |
| Recipe | MUST 15 + NICE ~6 + LOW (나머지 전부 JSON에) |
| ClassJob | MUST 14 + NICE 12 + LOW (나머지 전부 JSON에) |
| 필드명 정규화 | 결정론적 알고리즘 only, 오버라이드 테이블 없음 |
| 익명 필드 | 드롭 (해석 불가 기본값) — `exd-header.json`으로 복구 시도 |
| 파생 필드 | Recipe의 recipeLevel/maxQuality = Gleaner flatten |
| 보조 테이블 | BaseParam 이름 / ItemUICategory 이름은 summaries와 embed |
| raw 변화 대응 | Gleaner의 diff 리포트로 자동 감지 |
| Gleaner 역할 | pure parser + domain enricher + serializer 3레이어 (Phase C에서 구체화) |

---

## 7. 다음 단계

- [[data-contract]] (Phase C): UI 노출 관점에서의 타입 계약 설명 문서. 실제 타입의 진실은 Gleaner 생성 `.d.ts`.
- [[gleaner-redesign]] (Phase C): 필드명 정규화 알고리즘, RecipeLevelTable flatten, raw 변화 diff 리포트, 3레이어 구조.
- [[frontend-plan]] (Phase C): 엔티티 그래프 탐색 UX. 이 문서의 MUST/NICE 분류를 실제 컴포넌트/탭으로 매핑.
