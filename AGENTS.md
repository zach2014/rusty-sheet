# AGENTS.md — Rusty Sheet

A DuckDB extension (Rust) that reads Excel, WPS, and OpenDocument spreadsheets
directly from SQL. Exposes four table functions: `read_sheet`, `read_sheets`,
`analyze_sheet`, `analyze_sheets`.

## Build / Test / Lint

```bash
make configure          # first time only: create venv, fetch extension-ci-tools
make debug              # build debug + extension metadata
make release            # build release (LTO + stripped)
make test               # run tests in debug mode
```

- `make debug` produces `build/debug/extension/rusty-sheet/rusty-sheet.duckdb_extension`
- Load locally with `dduckdb -unsigned` then `LOAD 'build/debug/extension/rusty-sheet/rusty-sheet.duckdb_extension'`
- No `cargo fmt` / `cargo clippy` targets in the Makefile — run `cargo fmt` and `cargo clippy` manually as needed
- Rust edition **2024**; crate type is `cdylib`

## Architecture

```
SQL query
  │
  ▼
lib.rs ─── extension_entrypoint() registers 4 table functions
  │
  ▼
extension/ ─── parameter parsing (Param / NamedParam traits) + DuckDB data output
  │
  ▼
spreadsheet/ ─── open_spreadsheet() dispatches by file extension
  │              Spreadsheet trait → XlsxSpreadsheet | XlsSpreadsheet | XlsbSpreadsheet | OdsSpreadsheet
  │              Returns Vec<Sheet> → analyze_sheets() produces Vec<Table>
  ▼
helpers/ ─── low-level parsers: XML, ZIP, CFB, BIFF8, BIFF12, string encoding, UnifiedReader (local + remote)
```

### Module tree

| Module | Purpose |
|--------|---------|
| `src/lib.rs` | C API entry point (`rusty_sheet_init_c_api`), DuckDB extension registration |
| `src/extension/` | DuckDB table functions (`read_sheet`, `read_sheets`, `analyze_sheet`, `analyze_sheets`), SQL parameter traits, data chunk writing |
| `src/spreadsheet/` | Format-independent `Spreadsheet` trait + format readers (`xlsx.rs`, `xls.rs`, `xlsb.rs`, `ods.rs`), `Cell`, `Criteria`, `Sheet` chunked data model, `excel.rs` shared Excel logic |
| `src/helpers/` | Parsing primitives: `xml.rs` (quick-xml wrappers), `zip.rs`, `cfb.rs` (OLE2), `biff8.rs`/`biff12.rs` (binary Excel), `reader.rs` (UnifiedReader: local + HTTP/S3/GS/HF), `string.rs` (encoding) |
| `src/database/` | Data types: `Column`/`ColumnType`, `Table`, `Range` (Excel-style ranges), `ValueBridge` (DuckDB FFI value access) |
| `src/error.rs` | Central `RustySheetError` enum wrapping all sub-module errors + helper traits (`ResultMessage`, `ResultOptionChain`) |

### Data flow

1. SQL calls `read_sheet('file.xlsx', sheet='Sheet1')`
2. `ReadSheetTableFunction` parses parameters via `Param`/`NamedParam` traits
3. `open_spreadsheet()` matches file extension → constructs format-specific reader
4. Reader implements `Spreadsheet::read_sheets()` → returns `Vec<Sheet>` with cells organized in 2048-row chunks
5. For `read_sheet`: cells written directly to DuckDB `DataChunk` via `Inserter`
6. For `analyze_sheet`: `Spreadsheet::analyze_sheets()` auto-detects column types, returns `Vec<Table>`

## Key Files

| File | Role |
|------|------|
| `Cargo.toml` | Dependencies, crate config. Note: `libduckdb-sys` version must match DuckDB target |
| `Makefile` | Wraps DuckDB `extension-ci-tools`. Set `TARGET_DUCKDB_VERSION` for the DuckDB API version |
| `.github/workflows/MainDistributionPipeline.yml` | CI via DuckDB's shared pipeline — builds on push/PR, tests, publishes releases |
| `.cargo/config.toml` | Windows-only: statically link C runtime |
| `src/lib.rs` | Extension registration — add new table functions here |
| `src/error.rs` | All errors funnel through `RustySheetError` — add new `#[from]` variants for new sub-errors |
| `src/spreadsheet/mod.rs` | `open_spreadsheet()` dispatcher — add new format extensions here |
| `src/extension/mod.rs` | SQL parameter definitions — add new named parameters here |

## Coding Conventions

- **Visibility**: Everything is `pub(crate)` — no public API beyond the C FFI entry points
- **Error handling**: `thiserror` derive enums (one per module) aggregated into `RustySheetError` via `#[from]`. Helper traits `ResultMessage` (prefix context) and `ResultOptionChain` (chain `Ok(None)` fallbacks)
- **Traits**: `Spreadsheet` for format polymorphism; `Param<T>`/`NamedParam<T>` for type-safe SQL parameter extraction from DuckDB `BindInfo`
- **Commit style**: Conventional commits — `feat:`, `fix:`, `docs:`, `Release vX.Y.Z`. PRs from forks, merged via GitHub
- **Comments**: Doc comments on public API in `lib.rs`; inline comments in Chinese occasionally appear (e.g., `// 忽略空工作表`) — prefer English for new code
- **Testing**: Tests run through DuckDB SQL test runner (see `make test`). Test SQL files live in `/test/`
- **No unsafe in business logic**: `unsafe` confined to FFI boundary (`lib.rs`, `bridge.rs`)

## Git Workflow

- Branch: `main` (protected)
- PRs from forks, squash-merged with conventional commit subject
- Releases tagged as e.g. `v0.4.2`
- No branch naming convention observed beyond feature branches from contributors

## CI/CD

- **Trigger**: push to `main`, pull requests, manual `workflow_dispatch`
- **Pipeline**: DuckDB's `_extension_distribution.yml` reusable workflow
- **Builds**: Linux (amd64, arm64), macOS (amd64, arm64), Windows (amd64) — WASM excluded
- **DuckDB target**: v1.4.3 (set in CI YAML, must match `Cargo.toml` duckdb crate version)

## Tips for AI Agents

- **Adding a new file format**: implement `Spreadsheet` trait in `src/spreadsheet/`, add an arm to `open_spreadsheet()` in `mod.rs`, add any new helper modules under `src/helpers/`
- **Adding a new SQL function**: create a struct implementing `TableFunctionInfo` in `src/extension/`, register it in `lib.rs::extension_entrypoint()`, add parameters as `Param`/`NamedParam` impls in `mod.rs`
- **Adding a new parameter**: define a struct in `extension/mod.rs`, impl `NamedParam<T>` for it, and thread it through the relevant function's parameter struct
- **Cell type detection**: `CellType::parse_builtin_number_format_id` / `parse_custom_number_format` in `cell.rs` — Excel date/time format heuristics live here
- **Column type inference**: `ColumnType::detect()` in `column.rs` uses a specificity hierarchy: Boolean > BigInt > Double > Date > Time > Timestamp > Varchar
- **Chunked reading**: `Sheet` processes data in 2048-row chunks for memory efficiency — see `sheet.rs`
- **Remote files**: `UnifiedReader` (in `helpers/reader.rs`) transparently handles `http://`, `https://`, `s3://`, `gs://`, `hf://` — glob expansion only applies to local paths
- **Build issues**: if `make configure` fails, check Python 3 + venv are available; the Makefile is thin, real build logic is in `extension-ci-tools/makefiles/`
- **.gitignore** excludes `build/`, `target/`, `venv/` — extension binaries are never committed
