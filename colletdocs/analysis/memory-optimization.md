# rust-analyzer 메모리 최적화 분석

## 개요

rust-analyzer는 560K+ 라인, 36개 crate로 구성된 대규모 Rust LSP 서버입니다.
Salsa 기반 증분 계산 구조를 사용하며, 모든 소스 파일 텍스트, 파싱 결과, 매크로 확장 결과, 타입 추론 결과를 메모리에 캐시합니다.

---

## 🔴 Critical: Salsa LRU 캐시 비활성화됨

**파일**: `crates/ide-db/src/lib.rs` (218~251행)

```rust
pub fn update_base_query_lru_capacities(&mut self, _lru_capacity: Option<u16>) {
    // let lru_capacity = lru_capacity.unwrap_or(base_db::DEFAULT_PARSE_LRU_CAP);
    // base_db::FileTextQuery.in_db_mut(self).set_lru_capacity(DEFAULT_FILE_TEXT_LRU_CAP);
    // base_db::ParseQuery.in_db_mut(self).set_lru_capacity(lru_capacity);
    // hir::db::ParseMacroExpansionQuery.in_db_mut(self).set_lru_capacity(4 * lru_capacity);
    // hir::db::BorrowckQuery.in_db_mut(self).set_lru_capacity(base_db::DEFAULT_BORROWCK_LRU_CAP);
    // hir::db::BodyWithSourceMapQuery.in_db_mut(self).set_lru_capacity(2048);
}
```

**문제**: Salsa 마이그레이션 과정에서 **모든 LRU 캐시 설정이 주석 처리**됨.  
파일 텍스트, 파싱 결과, 매크로 확장 결과, borrowck 결과 등이 **무제한으로 메모리에 누적**되고 있음.

**영향도**: ⭐⭐⭐⭐⭐ (가장 큰 단일 메모리 낭비 원인)  
**해결**: Salsa 새 버전에 맞게 LRU 캐시 설정을 복원. 특히:
- `FileTextQuery`: 현재 활성 파일만 유지
- `ParseQuery`: 최근 파싱 결과만 캐시
- `ParseMacroExpansionQuery`: 매크로 확장 결과 제한
- `BorrowckQuery`: borrow check 결과 제한

---

## 🔴 Critical: 파일 텍스트 전체 상주

**파일**: `crates/base-db/src/lib.rs`

```rust
pub struct Files {
    files: Arc<DashMap<vfs::FileId, FileText, BuildHasherDefault<FxHasher>>>,
    source_roots: Arc<DashMap<SourceRootId, SourceRootInput, ...>>,
    file_source_roots: Arc<DashMap<vfs::FileId, FileSourceRootInput, ...>>,
}
```

- 열린 **모든 파일의 전체 텍스트**가 `DashMap`에 `Arc<str>`로 저장됨
- 라이브러리(표준 라이브러리 + crates.io 디펜던시) 파일까지 모두 상주
- 대형 프로젝트에서 수천 개의 파일이 동시에 메모리에 있음

**개선 방안**:
1. 라이브러리 파일 텍스트를 필요시에만 로드 (lazy loading)
2. 비활성 파일은 압축된 형태로 저장
3. 파일 텍스트를 LRU 캐시로 관리 (위 LRU 복원과 연계)

---

## 🟠 High: 매크로 확장 결과 무제한 캐싱

**파일**: `crates/hir-expand/src/db.rs`, `crates/tt/src/storage.rs`

```rust
fn parse_macro_expansion(
    db: &dyn ExpandDatabase,
    macro_file: MacroCallId,
) -> ExpandResult<(Parse<SyntaxNode>, Arc<ExpansionSpanMap>)>
```

- 매크로 확장 결과(`SyntaxNode` 트리 + `ExpansionSpanMap`)가 **Salsa에 무제한 캐시**됨
- `include!` 매크로, `serde` derive, `tokio` 매크로 등 대형 확장은 수 MB 소모
- `TopSubtree` (Token Tree)의 Span 압축은 이미 잘 되어 있으나, **파싱된 트리 자체**가 무거움

**개선 방안**:
1. LRU 캐시로 매크로 확장 결과 개수 제한
2. 자주 사용되지 않는 확장 결과는 `Arc` strong count 기반으로 자동 해제
3. `ExpansionSpanMap`을 필요시 재계산 가능하도록 분리

---

## 🟠 High: Expr/Pat Enum 크기 최적화 필요

**파일**: `crates/hir-def/src/hir.rs`

```rust
pub enum Expr {
    // ...
    InlineAsm(InlineAsm),  // 가장 큰 variant
    Block { id: Option<BlockId>, statements: Box<[Statement]>, tail: Option<ExprId>, label: Option<LabelId> },
    MethodCall { receiver: ExprId, method_name: Name, args: Box<[ExprId]>, generic_args: Option<Box<GenericArgs>> },
    RecordLit { path: Option<Box<Path>>, fields: Box<[RecordLitField]>, spread: RecordSpread },
    Match { expr: ExprId, arms: Box<[MatchArm]> },
    // ...
}
```

- `Expr` enum은 `InlineAsm` variant 때문에 크기가 큼
- `Arena<Expr>`에 수천~수만 개의 Expr이 저장됨
- padding으로 인해 실제 필요 크기보다 훨씬 큼

**개선 방안**:
1. `InlineAsm`을 `Box<InlineAsm>`으로 래핑 (이미 Box인지 확인 필요)
2. 드문 variant들을 Box로 래핑하여 enum 크기 축소
3. `Name`의 `ctx: ()` 필드 제거 (필요시 별도 구조체로 분리)

---

## 🟠 High: Symbol/Name 인터닝 최적화

**파일**: `crates/intern/src/symbol.rs`, `crates/hir-expand/src/name.rs`

```rust
// Symbol: TaggedArcPtr (thin pointer) - 이미 잘 최적화됨
// 하지만 Name에 불필요한 ctx 필드가 있음
pub struct Name {
    symbol: Symbol,
    ctx: (),  // ← 언제나 유닛, 불필요한 메모리
}
```

**개선 방안**:
1. `Name.ctx: ()` 제거 → `Name` 크기를 `Symbol`과 동일하게
2. 수백만 개의 `Name` 인스턴스가 각각 0바이트의 padding을 가질 수 있음
3. `Symbol` 시스템 자체는 이미 `TaggedArcPtr`로 잘 최적화됨

---

## 🟡 Medium: Hygiene/SyntaxContext 데이터

**파일**: `crates/span/src/hygiene.rs`, `crates/hir-expand/src/db.rs`

```rust
pub struct SyntaxContextData {
    outer_expn: Option<MacroCallId>,
    outer_transparency: Transparency,
    edition: Edition,
    parent: SyntaxContext,
    opaque: SyntaxContext,
    // ...
}
```

- 매크로 확장마다 새로운 `SyntaxContext`가 생성됨 (Salsa로 interned)
- 깊이 중첩된 매크로에서 연쇄적으로 생성됨
- `SyntaxContext` 체인이 길어지면 mark-and-sweep GC에서 순회 비용 증가

**개선 방안**:
1. 동일한 hygiene 컨텍스트를 공유하는 토큰들 그룹화
2. `SyntaxContextData` 필드 레이아웃 최적화 (크기 축소)
3. GC 빈도 조정으로 stale 컨텍스트 정리

---

## 🟡 Medium: ItemTree 중복 데이터

**파일**: `crates/hir-def/src/item_tree.rs`

```rust
pub struct ItemTree {
    top_level: Box<[ModItemId]>,
    top_attrs: AttrsOrCfg,
    attrs: FxHashMap<FileAstId<ast::Item>, AttrsOrCfg>,
    vis: ItemVisibilities,
    big_data: FxHashMap<FileAstId<ast::Item>, BigModItem>,
    small_data: FxHashMap<FileAstId<ast::Item>, SmallModItem>,
}
```

- 파일당 하나의 `ItemTree`가 Salsa에 캐시됨
- `attrs` 맵이 파일 내 모든 아이템의 속성을 보관
- 라이브러리 파일의 ItemTree는 절대 변경되지 않으나 영구 보관

**개선 방안**:
1. 라이브러리 ItemTree는 디스크에서 직렬화/역직렬화
2. `AttrsOrCfg`의 cfg 제거된 항목은 GC로 정리
3. `FxHashMap` → `Box<[(K, V)]>` 정렬 배열로 전환 (작은 맵의 경우 더 효율적)

---

## 🟡 Medium: SymbolIndex (fst) 메모리

**파일**: `crates/ide-db/src/symbol_index.rs`

- 작업 공간 전체의 심볼 검색을 위한 FST(Finite State Transducer) 인덱스
- 라이브러리당 하나의 FST + 로컬 파일당 하나의 FST 생성
- 대형 워크스페이스에서 수십~수백 개의 FST가 메모리에 상주

**개선 방안**:
1. 라이브러리 FST를 mmap으로 관리
2. 비활성 라이브러리의 FST lazy loading
3. FST 빌드 후 중간 데이터 해제 확인

---

## 🟢 Low: 이미 잘 최적화된 부분

| 영역 | 최적화 상태 |
|------|------------|
| **Token Tree Span** | `TopSubtree`에서 32/64/96비트 3단계 압축 저장 ✅ |
| **ThinVec 사용** | 자주 비어있는 Vec에 `ThinVec` 적용 (macro_calls 등) ✅ |
| **Symbol 인터닝** | `TaggedArcPtr`로 static/interned 구분, 포인터 크기만 차지 ✅ |
| **FxHashMap** | 프로젝트 전반에 `rustc_hash::FxHashMap` 사용 ✅ |
| **Triomphe Arc** | `std::sync::Arc` 대신 zero-cost `triomphe::Arc` 사용 ✅ |
| **GC 시스템** | mark-and-sweep + 병렬 sweep으로 interned 값 정리 ✅ |
| **ExprId/PatId** | 32비트 인덱스로 Arena 참조 ✅ |

---

## 📊 권장 우선순위

| 순위 | 항목 | 예상 효과 | 난이도 |
|------|------|----------|--------|
| 1 | **LRU 캐시 복원** | 메모리 30-50% 절감 | 중간 (Salsa API 이해 필요) |
| 2 | **파일 텍스트 lazy loading** | 대형 프로젝트에서 수백 MB 절감 | 높음 |
| 3 | **매크로 확장 캐시 제한** | 메모리 20-30% 절감 | 중간 |
| 4 | **Name.ctx 제거** | 수백만 Name 인스턴스에 미치는 영향 | 낮음 |
| 5 | **Expr enum 크기 최적화** | Arena 전체 크기 축소 | 낮음 |
| 6 | **SyntaxContext 최적화** | 깊은 ���크로 중첩 시 효과 | 중간 |

---

## 🛠 즉시 실행 가능한 작업

### 1. Name.ctx 제거
```rust
// Before
pub struct Name {
    symbol: Symbol,
    ctx: (),
}

// After
pub struct Name {
    symbol: Symbol,
}
```
이것만으로도 수백만 개의 `Name` 인스턴스에서 메모리 절약.

### 2. LRU 캐시 복원 (Salsa v3 API 확인 필요)
`crates/ide-db/src/lib.rs`의 주석 처리된 LRU 설정을 새 Salsa API에 맞게 복원.

### 3. Files DashMap 크기 모니터링
`Files` 구조체에 현재 보관 중인 파일 수/총 바이트 추적 기능 추가.
