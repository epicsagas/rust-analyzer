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

---

## 🟠 High: InferenceResult — 함수당 ~15개 맵/배열 영구 캐시

**파일**: `crates/hir-ty/src/infer.rs`

```rust
pub struct InferenceResult {
    method_resolutions: FxHashMap<ExprId, (FunctionId, StoredGenericArgs)>,
    field_resolutions: FxHashMap<ExprId, Either<FieldId, TupleFieldId>>,
    variant_resolutions: FxHashMap<ExprOrPatId, VariantId>,
    assoc_resolutions: FxHashMap<ExprOrPatId, (CandidateId, StoredGenericArgs)>,
    tuple_field_access_types: ThinVec<StoredTys>,

    type_of_expr: ArenaMap<ExprId, StoredTy>,
    type_of_pat: ArenaMap<PatId, StoredTy>,
    type_of_binding: ArenaMap<BindingId, StoredTy>,
    type_of_type_placeholder: FxHashMap<TypeRefId, StoredTy>,
    type_of_opaque: FxHashMap<InternedOpaqueTyId, StoredTy>,
    type_mismatches: Option<Box<FxHashMap<ExprOrPatId, TypeMismatch>>>,
    diagnostics: ThinVec<InferenceDiagnostic>,
    expr_adjustments: FxHashMap<ExprId, Box<[Adjustment]>>,
    pat_adjustments: FxHashMap<PatId, Vec<StoredTy>>,
    binding_modes: ArenaMap<PatId, BindingMode>,
    closure_info: FxHashMap<InternedClosureId, (Vec<CapturedItem>, FnTrait)>,
    mutated_bindings_in_closure: FxHashSet<BindingId>,
    coercion_casts: FxHashSet<ExprId>,
    // ...
}
```

**핵심 통계**: `#[salsa::tracked(returns(ref))]`로 **74개** 함수가 영구 캐시되며, 그중 **LRU가 설정된 것은 단 6개**뿐.

**문제**:
- `InferenceResult`는 함수(`DefWithBodyId`)당 하나씩 생성되며 `returns(ref)`로 영구 캐시
- 대형 프로젝트(수만 함수)에서 수만 개의 `InferenceResult`가 동시에 메모리에 존재
- 각 결과에 `FxHashMap` 8개 + `ArenaMap` 3개 + `ThinVec` 1개 + `FxHashSet` 1개 포함
- `StoredGenericArgs`는 인터닝된 슬라이스 참조이나, 맵 오버헤드 자체가 큼

**개선 방안**:
1. `InferenceResult`를 LRU 캐시로 전환 (예: 4096~8192)
2. 드문 필드(`type_of_type_placeholder`, `type_of_opaque`)를 `Option<Box<...>>`로 지연 할당
3. `expr_adjustments`를 `ThinVec` 기반으로 전환 (대부분의 expr은 adjustment 없음)
4. `closure_info`를 별도 LRU 쿼리로 분리

---

## 🟠 High: MirBody + BorrowckResult — MIR 당 수십 KB

**파일**: `crates/hir-ty/src/mir.rs`, `crates/hir-ty/src/mir/borrowck.rs`

```rust
pub struct MirBody {
    projection_store: ProjectionStore,   // FxHashMap<ProjectionId, Box<[PlaceElem]>>
                                          // + FxHashMap<Box<[PlaceElem]>, ProjectionId> (역방향)
    basic_blocks: Arena<BasicBlock>,     // 각 블록에 Vec<Statement> + Terminator
    locals: Arena<Local>,
    binding_locals: ArenaMap<BindingId, LocalId>,
    param_locals: Vec<LocalId>,
    closures: Vec<InternedClosureId>,
}

pub struct BorrowckResult {
    mir_body: Arc<MirBody>,
    mutability_of_locals: ArenaMap<LocalId, MutabilityReason>,
    moved_out_of_ref: Vec<MovedOutOfRef>,
    partially_moved: Vec<PartiallyMoved>,
    borrow_regions: Vec<BorrowRegion>,
}
```

**문제**:
- MIR은 함수마다 `Arena<BasicBlock>` + `ProjectionStore` 이중 맵 포함
- `BorrowckResult`는 `Arc<MirBody>` + 추가 Vec 3개 보유
- `mir_body`와 `borrowck`이 `Arc<MirBody>`를 **공유하지만**, 각각 Salsa에 별도 캐시
- `projection_store`가 양방향 맵(FxHashMap 2개)으로 projection당 2배 메모리 사용

**개선 방안**:
1. MIR/Borrowck을 LRU 캐시로 전환 (IDE에서 모든 함수의 MIR이 동시에 필요하지 않음)
2. `ProjectionStore`에서 역방향 맵 제거 또는 `HashMap<Box<[T]>, Id>` → `hashbrown::HashSet` 기반 최적화
3. `BorrowckResult`의 Vec 필드를 `ThinVec`으로 전환

---

## 🟠 High: ItemScope — 모듈당 3개 FxIndexMap + 3개 FxHashMap

**파일**: `crates/hir-def/src/item_scope.rs`

```rust
pub struct ItemScope {
    types: FxIndexMap<Name, TypesItem>,
    values: FxIndexMap<Name, ValuesItem>,
    macros: FxIndexMap<Name, MacrosItem>,
    unresolved: FxHashSet<Name>,

    declarations: ThinVec<ModuleDefId>,
    impls: ThinVec<(ImplId, bool)>,
    // ...

    use_imports_types: FxHashMap<ImportOrExternCrate, ImportOrDef>,
    use_imports_values: FxHashMap<ImportOrGlob, ImportOrDef>,
    use_imports_macros: FxHashMap<ImportOrExternCrate, ImportOrDef>,

    legacy_macros: FxHashMap<Name, SmallVec<[MacroId; 2]>>,
    derive_macros: FxHashMap<AstId<ast::Adt>, SmallVec<[DeriveMacroInvocation; 1]>>,
}
```

**문제**:
- **모듈마다** `FxIndexMap` 3개 + `FxHashMap` 5개 + `FxHashSet` 1개 = **맵 9개**
- 대형 프로젝트에서 수천 개의 모듈 → 수만 개의 해시맵 인스턴스
- `FxIndexMap`은 `IndexMap` 기반으로 `HashMap`보다 메모리 오버헤드가 큼
- `use_imports_*` 맵은 대부분의 모듈에서 거의 비어있음

**개선 방안**:
1. 자주 비어있는 맵(`use_imports_*`, `legacy_macros`, `derive_macros`)을 `Option<Box<FxHashMap>>`으로 지연 할당
2. `FxIndexMap` 3개(types/values/macros)를 단일 `FxIndexMap<Name, PerNs>`로 병합 검토
3. `ImportOrDef` enum의 크기 확인 후 필요시 Box 래핑

---

## 🟡 Medium: FunctionSignature — 모든 함수에 ExpressionStore 포함

**파일**: `crates/hir-def/src/signatures.rs`

```rust
pub struct FunctionSignature {
    pub name: Name,
    pub generic_params: GenericParams,         // Arena + Arena + Box<[WherePredicate]>
    pub store: ExpressionStore,                // types Arena + lifetimes Arena + Option<Box<ExpressionOnlyStore>>
    pub params: Box<[TypeRefId]>,
    pub ret_type: Option<TypeRefId>,
    pub abi: Option<Symbol>,
    pub flags: FnFlags,
}
```

**문제**:
- **모든 함수, 구조체, 열거형, trait, impl** 마다 Signature이 캐시됨
- `FunctionSignature` 안에 `ExpressionStore`이 포함 — 타입 참조를 위한 Arena
- `returns(ref)` 또는 `returns(deref)`로 영구 캐시, LRU 없음
- `GenericParams`에도 `Arena<TypeOrConstParamData>` + `Arena<LifetimeParamData>` 포함
- Signature 수 = 프로젝트 내 정의 수 (수만~수십만)

**개선 방안**:
1. `with_source_map` 변형을 LRU 캐시로 전환 (IDE에서 source map이 필요한 경우는 제한적)
2. `GenericParams`의 `where_predicates: Box<[WherePredicate]>`를 필요시 재계산

---

## 🟡 Medium: Binding의 Name 중복 — Arena에 수천 개

**파일**: `crates/hir-def/src/hir.rs`

```rust
pub struct Binding {
    pub name: Name,                    // Symbol + ctx: ()
    pub mode: BindingAnnotation,       // 1 byte
    pub problems: Option<BindingProblems>, // 1 byte + padding
    pub hygiene: HygieneId,            // SyntaxContext wrapper
}
```

**문제**:
- `ExpressionOnlyStore`의 `Arena<Binding>`에 함수당 수십~수백 개의 Binding 저장
- 각 `Binding`이 `Name` 필드 보유 → `Name`은 `Symbol`(8바이트) + `ctx: ()`(0바이트 + padding)
- 같은 이름의 변수가 여러 번 나타나면 `Symbol`은 인터닝되어 공유되지만, `Name` 래퍼는 매번 새로 생성
- `BindingAnnotation` + `Option<BindingProblems>` = 2바이트이지만 padding으로 인해 낭비 발생 가능

**개선 방안**:
1. `Name.ctx: ()` 제거 (이미 위에서 언급)
2. `Binding.problems`를 `BindingProblems` 대신 `u8` 비트플래그로 표현
3. `Binding` 필드 순서 재배치로 padding 최소화

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

---

## 🟡 Medium: CrateData / Env — crate당 String 맵

**파일**: `crates/base-db/src/input.rs`

```rust
pub struct CrateData<Id> {
    pub root_file_id: FileId,
    pub dependencies: Vec<Dependency<Id>>,   // 각 Dependency에 CrateName(Symbol)
    pub origin: CrateOrigin,
    pub crate_attrs: Box<[Box<str>]>,        // crate-level 속성 문자열들
    pub is_proc_macro: bool,
    pub proc_macro_cwd: Arc<AbsPathBuf>,
}

pub struct Env {
    entries: FxHashMap<String, String>,       // 환경변수 맵
}

pub struct ExtraCrateData {
    pub version: Option<String>,
    pub display_name: Option<CrateDisplayName>,
    pub potential_cfg_options: Option<CfgOptions>,
}
```

**문제**:
- crate마다 `Env`의 `FxHashMap<String, String>` 보유 — 키/값이 개별 `String` 할당
- `CrateData.dependencies: Vec<Dependency>` — crate 수 × 평균 디펜던시 수
- `crate_attrs: Box<[Box<str>]>` — 속성 문자열이 개별 힙 할당

**개선 방안**:
1. `Env.entries`의 키를 `Arc<str>` 또는 interned `Symbol`로 전환 (자주 반복되는 환경변수명)
2. `crate_attrs`를 `Arc<str>` 하나로 연결 저장

---

## 🟡 Medium: PackageData (project-model) — String 남발

**파일**: `crates/project-model/src/cargo_workspace.rs`

```rust
pub struct PackageData {
    pub version: semver::Version,
    pub name: String,
    pub repository: Option<String>,
    pub manifest: ManifestPath,
    pub description: Option<String>,
    // ...
}

pub struct TargetData {
    pub name: String,
    pub root: AbsPathBuf,
    pub required_features: Vec<String>,  // ← Vec<String>!
}
```

**문제**:
- `PackageData`에 `String` 필드 3개, `TargetData`에 `String` + `Vec<String>`
- `required_features: Vec<String>` — 각 feature 이름이 개별 `String` 할당
- 전체 크레이트 수 × 평균 타겟 수 × 필드 수만큼의 `String` 할당

**개선 방안**:
1. `PackageData.name`, `TargetData.name`을 `Arc<str>`로 전환 (여러 곳에서 참조)
2. `required_features`를 `Box<[Arc<str>]>`로 전환

---

## 🟡 Medium: Docs 문자열 — 아이템당 연결된 doc 문자열

**파일**: `crates/hir-def/src/attrs/docs.rs`

```rust
pub struct Docs {
    docs: String,                                // 연결된 전체 doc 문자열
    docs_source_map: Vec<DocsSourceMapLine>,     // 소스 맵핑 정보
    outline_mod: Option<(HirFileId, usize)>,
    inline_file: HirFileId,
    prefix_len: TextSize,
    inline_inner_docs_start: Option<TextSize>,
}
```

**문제**:
- 문서화된 아이템마다 `Docs` 구조체가 생성되어 `Box<Docs>`로 반환
- `docs: String`은 해당 아이템의 모든 `#[doc = "..."]`을 연결한 전체 문자열
- 표준 라이브러리의 수천 개 아이템에 대해 긴 doc 문자열이 메모리에 상주
- `docs_source_map: Vec<DocsSourceMapLine>` — doc → 소스 오프셋 매핑 (hover 기능용)

**개선 방안**:
1. `Docs`를 LRU 캐시로 관리 (이미 `attrs.rs`에서 `lru = 250`이 적용된 `documentation` 쿼리 있음)
2. 라이브러리 아이템의 doc은 필요시에만 로드
3. `docs_source_map`을 hover 요청이 있을 때만 생성하는 lazy 패턴

---

## 🟡 Medium: Proc-macro dylib 캐시

**파일**: `crates/proc-macro-srv/src/lib.rs`, `crates/proc-macro-srv/src/dylib.rs`

```rust
struct ProcMacroSrv {
    expanders: Mutex<HashMap<Utf8PathBuf, Arc<dylib::Expander>>>,
}

struct ProcMacroLibrary {
    proc_macros: &'static ProcMacros,
    _lib: Library,   // 로드된 동적 라이브러리 전체
}
```

**문제**:
- 로드된 모든 proc-macro dylib가 `HashMap`에 영구 캐시됨
- 각 `Expander`는 전체 `.so`/`.dll` 파일을 메모리에 매핑한 상태 유지
- 대형 프로젝트(수십 개 proc-macro crate)에서 누적 메모리가 수백 MB에 달할 수 있음

**개선 방안**:
1. 사용하지 않는 dylib를 reference counting으로 자동 언로드
2. TTL 기반 만료 (일정 시간 미사용 시 언로드)
3. dylib 매핑을 `mmap` 기반으로 전환하여 OS가 필요시 페이지 인/아웃하게

---

## 🟡 Medium: ProjectionStore — MIR의 이중 해시맵

**파일**: `crates/hir-ty/src/mir.rs`

```rust
pub struct ProjectionStore {
    id_to_proj: FxHashMap<ProjectionId, Box<[PlaceElem]>>,
    proj_to_id: FxHashMap<Box<[PlaceElem]>, ProjectionId>,
}
```

**문제**:
- 모든 projection에 대해 **동일한 데이터를 2개의 맵에 양방향 저장**
- `Box<[PlaceElem]>`이 두 맵에 각각 별도로 할당됨
- projection이 많은 함수(복잡한 패턴 매칭, 다차원 배열 접근)에서 메모리 2배 소모

**개선 방안**:
1. 정방향 맵만 유지 + 역방향은 필요시 선형 검색 또는 전용 인덱스
2. `Box<[PlaceElem]>`을 interned slice로 전환하여 두 맵이 동일한 스토리지 공유

---

## 🟡 Medium: TraitImpls — crate당 FxHashMap

**파일**: `crates/hir-ty/src/method_resolution.rs`

```rust
pub struct TraitImpls {
    map: FxHashMap<TraitId, OneTraitImpls>,
}

struct OneTraitImpls {
    non_blanket_impls: FxHashMap<SimplifiedType, (Box<[ImplId]>, Box<[BuiltinDeriveImplId]>)>,
    blanket_impls: Box<[ImplId]>,
}
```

**문제**:
- crate마다 `TraitImpls` 생성 (`#[salsa::tracked(returns(ref))]`)
- `map`에 trait 수만큼의 엔트리, 각 엔트리에 또 `FxHashMap`이 중첩
- 전체 프로젝트의 trait impl 데이터가 영구 캐시

**개선 방안**:
1. 라이브러리 `TraitImpls`는 디스크에 직렬화하여 lazy load
2. `OneTraitImpls`의 `non_blanket_impls`를 `Box<[(SimplifiedType, ...)]>` 정렬 배열로 전환

---

## 🟡 Medium: ImportMap — crate당 FST + FxIndexMap

**파일**: `crates/hir-def/src/import_map.rs`

```rust
pub struct ImportMap {
    item_to_info_map: FxIndexMap<ItemInNs, (SmallVec<[ImportInfo; 1]>, IsTraitAssocItem)>,
    importables: Vec<(ItemInNs, u32)>,
    fst: fst::Map<Vec<u8>>,
}
```

**문제**:
- crate마다 `ImportMap`이 `#[salsa::tracked(returns(ref))]`로 영구 캐시
- `fst::Map<Vec<u8>>` — FST 바이트코드 전체가 메모리 상주
- `item_to_info_map` — 모든 공개 아이템에 대한 메타데이터

**개선 방안**:
1. 라이브러리 `ImportMap`의 FST를 mmap으로 관리
2. 라이브러리 `ImportMap` 전체를 lazy loading

---

## 🟡 Medium: lints.rs — 23K 라인 정적 데이터

**파일**: `crates/ide-db/src/generated/lints.rs`

```rust
pub struct Lint {
    pub label: &'static str,
    pub description: &'static str,
    pub default_severity: Severity,
    pub warn_since: Option<Edition>,
    pub deny_since: Option<Edition>,
}
```

- 23,610 라인, ~1,795개의 `Lint` 인스턴스 + `LintGroup` + `FEATURES`
- `&'static str`이므로 바이너리에 포함 → 런타임 메모리가 아닌 **바이너리 크기**에 영향
- 런타임에는 참조만 사용하므로 메모리 낭비는 아님
- 다만, 런타임에 이 데이터를 가��/필터링하여 새 `Vec`/`String`을 생성하는 경우 추가 할당 발생 가능

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
| 1 | **Salsa LRU 복원 (68개 무제한 → 제한)** | 메모리 30-50% 절감 | 중간 |
| 2 | **InferenceResult LRU + 지연 할당** | 함수 수 × ~KB 절감 | 중간 |
| 3 | **MirBody + BorrowckResult LRU** | MIR 미사용 함수 메모리 해제 | 낮음 |
| 4 | **ItemScope 빈 맵 지연 할당** | 모듈 수 × 9개 맵 → 실제 사용분만 | 낮음 |
| 5 | **파일 텍스트 lazy loading** | 대형 프로젝트에서 수백 MB 절감 | 높음 |
| 6 | **FunctionSignature LRU** | 함수 수 × Signature 크기 | 낮음 |
| 7 | **Name.ctx 제거** | 수백만 Name 인스턴스 | 낮음 |
| 8 | **Expr enum 크기 최적화** | Arena 전체 크기 축소 | 낮음 |
| 9 | **Binding 필드 재배치** | Arena<Binding> 크기 미세 최적화 | 낮음 |
| 10 | **Docs / ImportMap lazy loading** | 라이브러리 doc 문자열 지연 로드 | 중간 |
| 11 | **ProjectionStore 단방향 맵** | MIR projection 메모리 50% 절감 | 중간 |
| 12 | **Proc-macro dylib 자동 언로드** | 수십 개 proc-macro crate에서 효과 | 중간 |
| 13 | **SyntaxContext 최적화** | 깊은 매크로 중첩 시 효과 | 중간 |
| 14 | **PackageData String → Arc<str>** | project-model 메모리 절감 | 낮음 |
| 15 | **TraitImpls lazy loading** | 라이브러리 trait impl 디스크 캐시 | 높음 |

---

## 📈 현재 LRU 설정 현황 (핵심 통계)

| 구분 | 개수 |
|------|------|
| `#[salsa::tracked(returns(ref))]` **총 함수** | **74개** |
| 그중 **LRU 설정 있음** | **6개** (8%) |
| 그중 **LRU 없음** (무제한 캐시) | **68개** (92%) |
| `#[salsa::tracked]` (returns 없음) | **31개** |
| `#[salsa_macros::interned]` (인터닝 타입) | **8개** |

### 현재 LRU가 설정된 6개 함수
| 함수 | LRU 크기 | 파일 |
|------|----------|------|
| `parse_macro_expansion` | 1024 | `hir-expand/src/db.rs` |
| `EditionedFileId::parse` | 128 | `base-db/src/editioned_file_id.rs` |
| `Body::with_source_map` | 512 | `hir-def/src/expr_store/body.rs` |
| `fields_attrs_query` | 250 | `hir-def/src/attrs.rs` |
| `attrs` (per owner) | 250 | `hir-def/src/attrs.rs` |
| `attrs` (per variants) | 50 | `hir-def/src/attrs.rs` |

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

### 4. ItemScope 빈 맵 지연 할당
```rust
// Before
use_imports_types: FxHashMap<ImportOrExternCrate, ImportOrDef>,

// After
use_imports_types: Option<Box<FxHashMap<ImportOrExternCrate, ImportOrDef>>>,
```

### 5. InferenceResult 희귀 필드 지연 할당
```rust
// Before
type_of_type_placeholder: FxHashMap<TypeRefId, StoredTy>,

// After
type_of_type_placeholder: Option<Box<FxHashMap<TypeRefId, StoredTy>>>,
```
