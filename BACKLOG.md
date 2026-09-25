# Backlog — staylorx/clean_architecture_workspace

Open/pending items only. Decisions taken are recorded in `CHANGELOG.md`, never here.

- Repo: `staylorx/clean_architecture_workspace`, default branch `master`, HEAD `12e164bc8dec02b6145b64365217a73eb2843596` ("Refactor engines and enhance action capabilities").
- Toolchain used for evidence: Dart 3.13.1 / Flutter 3.47.1 on the Windows windesk lane.
- Sources: `REPORT_build_audit_windows.md` (task t_74a25be6) and `DEVIATIONS_clean_architecture_workspace.md` (task t_cfbac429); standards side `staylorx/dart-flutter-bible` @ `d4d2ff1`.
- Written by task t_af9c9fc1. Nothing in sections A or B has been fixed yet.

Current state, from real runs at HEAD `12e164bc` (verbatim exit codes):

    flutter pub get                                          -> exit 0   (dirty: pubspec.lock + 3 analysis_options.yaml)
    dart analyze                                             -> exit 3   (3825 issues: 2452 error / 235 warning / 1138 info)
    dart analyze --fatal-infos                               -> exit 3
    packages/architecture_lints dart test                    -> exit 1   (+242 -115; 79/115 test files fail to load)
    packages/clean_architecture_kit dart test                -> exit 1   (+0 -9; 9/9 test files fail to load)
    packages/clean_architecture_lints_experimental dart test -> exit 1   (+24 -54; 54/55 test files fail to load)
    examples/clean_feature_first dart run custom_lint        -> exit 0   ("No issues found!")

Issue totals by package: `clean_architecture_lints_experimental` 2253, `architecture_lints` 1149 (all 685 of its
errors sit in `test/`), `clean_architecture_kit` 328 (broken public entry point), `examples/clean_feature_first` 95.

## A. Build & analysis problems (open)

- **A1 — Refactor `12e164bc` renamed `lib/src` directories without updating imports: 122 unresolvable import URIs.**
  `src/lints` -> `src/lints_old`, `src/utils` -> `src/utils_old`; `src/analysis`/`src/models`/`src/actions`/`src/config`
  are gone and their code moved under `src/lints/`, `src/schema/`, `src/engines/`, `src/utils/`. Most frequent misses:
  74x `package:architecture_lints/src/analysis/layer_resolver.dart`,
  53x `package:architecture_lints/src/analysis/arch_component.dart`,
  19x `package:clean_architecture_kit/src/utils/layer_resolver.dart`,
  19x `package:architecture_lints/src/utils/config/config_keys_old.dart`,
  17x `package:architecture_lints/src/models/configs/architecture_config.dart`,
  13x `.../utils/extensions/json_map_extension.dart`, 13x `.../utils/extensions/iterable_extension.dart`,
  10x `package:clean_architecture_kit/src/utils/naming_utils.dart`, 10x `.../utils/nlp/naming_utils.dart`.
  *Fix:* repair the import URIs (or revert the rename) and delete the `*_old` generation.

- **A2 — `clean_architecture_kit` does not compile at all (public entry point broken).**
  `packages/clean_architecture_kit/lib/clean_architecture_kit.dart:2:8` imports `src/lints/data_source_purity.dart`
  (only `src/lints_old/data_source_purity.dart` exists) — same for 15 more `src/lints/*` imports, plus line 18
  `src/utils/layer_resolver.dart` (only `src/utils_old/layer_resolver.dart` exists). 328 issues; real test error:
  `test/clean_architecture_kit_test.dart:3:8: Error when reading 'lib/src/lints/data_source_purity.dart':
  The system cannot find the path specified`.
  *Fix:* point the barrel at the new layout, or restore the old paths.

- **A3 — `architecture_lints` is clean in `lib/` but its whole `test/` tree is stale (685 errors).**
  Suite `+242 -115`, 79 of 115 test files fail to load. Examples:
  `test/helpers/analyzer_test_utils.dart:9:8` uri_does_not_exist on `src/analysis/layer_resolver.dart`;
  `test/helpers/test_data.dart:54:10` undefined `ArchitectureConfig`;
  `test/src/actions/code_generator_test.dart:45:21` undefined `WriteStrategy`;
  `test/src/utils_old/syntax_builder_test.dart:87:29` undefined `SyntaxBuilder`.
  *Fix:* moved symbols (`WriteStrategy` -> `lib/src/schema/enums/write_strategy.dart:5`, `ArchitectureConfig` ->
  `lib/src/schema/config/architecture_config.dart:20`) need import updates; `ArchComponent` needs restoring (A4).

- **A4 — `ArchComponent` was deleted, not moved: referenced 53x by tests, exists nowhere in the repo.**
  `test/src/analysis/arch_component_test.dart:10:16: Undefined name 'ArchComponent'`.
  *Fix:* restore it from git history (`git log --diff-filter=D`) or drop/rewrite the 53 referencing tests — owner call.

- **A5 — `clean_architecture_lints_experimental`: public entry point (77 errors) and tests both stale.**
  `lib/clean_architecture_lints.dart` imports `src/lints_old/...` through `package:` URIs
  (`implementation_imports` + `uri_does_not_exist` + `depend_on_referenced_packages`); next worst are
  `lib/src/models/configs/architecture_config.dart` (36 errors) and `lib/src/analysis/layer_resolver.dart` (41 errors).
  54 of 55 test files fail to load, e.g.
  `test/src/analysis/arch_component_test.dart:3:8: Error when reading
  '../architecture_lints/lib/src/analysis/arch_component.dart': The system cannot find the path specified`.

- **A6 — Committed junk in `examples/clean_feature_first/lib/`.**
  A materialised mustache template `lib/features/auth/data/models/source.name.snakeCase_model.dart` (62 errors:
  `uri_with_interpolation`, `undefined_identifier` on `source`, `invalid_constructor_name`, ...) with the same shape
  as the tracked `templates/*.mustache` files, plus 12 `*.violations.dart` scratch/evidence files (`model.`,
  `repository.`, `entity.`, `usecase.`, `port.`, `login_screen.`, `orphan_logic.` ...).
  `login_screen.violations.dart:5:8` points at `.../data/repositories/auth_repository_impl.dart`, which does not exist.
  *Fix:* delete all 13; demo fixtures belong in `test/`.

- **A7 — Undeclared dev dependency `flutter_test` breaks 7 `architecture_lints` test files.**
  `test/src/config/parsing/package_path_resolver_test.dart`, `.../yaml_merger_test.dart`,
  `.../lints/boundaries/logic/component_logic_test.dart`, `.../lints/consistency/rules/orphan_file_rule_test.dart`,
  `.../lints/members/logic/member_logic_test.dart`, `.../lints/metadata/logic/annotation_logic_test.dart`,
  `.../lints/usages/logic/usage_logic_test.dart` import `package:flutter_test/flutter_test.dart`, but
  `grep -rn flutter_test --include=pubspec.yaml .` is empty.
  *Fix:* add `flutter_test: {sdk: flutter}` to that package's dev_dependencies, or switch those tests to `package:test`.

- **A8 — `packages/storage.json` is a tracked 5-byte non-UTF8 junk file** (`ff 0f 6f 7b 00`), not a Dart source and
  not a package. *Fix:* `git rm packages/storage.json`.

- **A9 — The lint gate contradicts itself.** `dart analyze` emits 5 real `arch_*` findings in
  `examples/clean_feature_first` (e.g. `lib/core/entity/entity.dart:3:1` arch_orphan_file,
  `lib/core/error/failures.dart:5:16` arch_naming_pattern) and logs
  `An error occurred while executing an analyzer plugin: ... Unknown request: analysis.setAnalysisRoots` (twice),
  while `dart run custom_lint` in the same directory prints `No issues found!` and exits 0.
  `examples/clean_feature_first/analysis_options.yaml` carries `custom_lint: {debug: true}`.
  *Fix:* settle on one gate (the melos `analyze` script uses custom_lint), fix the plugin crash, then trust that one.

- **A10 — `flutter pub get` rewrites tracked files on every run:** `pubspec.lock` (bumps `characters`,
  `material_color_utilities`, `meta`, `vector_math`) plus an `analyzer: exclude:` block re-inserted into
  `examples/clean_feature_first/analysis_options.yaml`, `packages/architecture_lints/analysis_options.yaml` and
  `packages/clean_architecture_lints_experimental/analysis_options.yaml`. These 4 files were found dirtied by the
  audit, were not committed, and were reverted before the docs commit. *Fix:* decide whether the tooling-managed
  config belongs in git; if yes, commit the regenerated form once and gate it.

- **A11 — No CI exists:** `.github/` is absent, so `dart analyze --fatal-infos --fatal-warnings` + `dart test`
  never run on a PR. (Cross-listed as the §09 checkbox-10 item.)

- **A12 — Zero tests in two members:** `packages/clean_architecture_core/test/` and
  `examples/clean_feature_first/test/` do not exist (core declares `test: ^1.26.0` but ships 0 test files).

- **A13 — Codegen deps declared but absent:** 0 `*.freezed.dart`, 0 `*.g.dart`, 0 `*.mapper.dart` and no
  `build.yaml` anywhere; the example's `dart_mappable` and freezed-family deps are unused by real sources
  (see the §07 items below). *Fix:* drop the unused deps or wire the codegen.

## B. Deviations from `staylorx/dart-flutter-bible` @ `d4d2ff1` (open)

Every deviation recorded by task t_cfbac429, reproduced verbatim (104 `Deviation:` items + 1 `Applicability:`
item). These are **flag-for-review**, not fix orders: this repo is a linter/codegen toolkit, not an application
built on the bible's bulls-eye, so several §01–§08 application rules have no target here.

### §00 — Compact (bot ingest blob)
- Deviation: `docs/00-compact.md` (bible side) - no bot blob and therefore no regeneration rule is present or enforceable in this repo; nothing in the code side derives from or checks the blob. [§00] (Informational; no code-side artifact exists to diverge.)

### §01 — Architecture (The Bulls-Eye)
- Deviation: `examples/clean_feature_first/lib/features/auth/domain/ports/auth_port.dart:8-15` - the `AuthPort` repository contract has exactly ONE adapter (`DefaultAuthRepository`); the bible's At-Least-Two-Repository-Adapter Rule and the shared contract suite are unmet, so the contract is whatever that one adapter does. [§01]
- Deviation: `examples/clean_feature_first/lib/features/auth/domain/entities/user.dart:6-11` - `User extends Entity` with plain `final` fields and a `const` ctor but no `equatable`/`props`; §01 law 2 requires value equality, and `equatable` appears nowhere in the repo (0 occurrences). [§01]
- Deviation: `packages/clean_architecture_core/lib/src/base/entity.dart:7-10` - the core `Entity` base carries no equality contract either, so every consumer entity inherits the gap. [§01]
- Deviation: `examples/clean_feature_first/lib/core/utils/service_locator.dart:1-14` - a global service locator (`getIt`/`locator`/`serviceLocator`) lets any code reach concrete implementations without going UI -> use case -> contract, breaking one-direction flow. [§01, cross-ref §11]
- Deviation: `examples/clean_feature_first/lib/core/usecase/usecase.dart:9-24` and `.../domain/usecases/request_login.dart:9-19` - the use-case seam takes a single positional container (`UnaryUsecase<ReturnType, ParameterType>` fed with a `_LoginParams` record) instead of discrete business inputs; §01's API-surface rule forbids cargo objects. [§01, cross-ref §04]
- Deviation: `examples/clean_feature_first/lib/features/auth/domain/ports/auth_port.dart:9` - `login(String username, String password)` uses positional parameters; §01/§02 require named parameters (sole exception a single positional `ref` or `message`). Same shape on `currentUser`. [§01, cross-ref §02]
- Deviation: `examples/clean_feature_first/lib/core/error/failures.dart:5-20` - `Failure` is an `abstract class` with non-`sealed`/non-`final` concrete leaves that each carry a display `String message`; §01 law 4 wants failures as typed values with exhaustive switches, and §08 forbids display strings in domain failures. [§01, cross-ref §04, §08]
- Deviation: `packages/clean_architecture_lints_experimental/lib/...` (200 import sites, e.g. `lib/clean_architecture_lints.dart:6`) - the package's public entry point reaches into `package:architecture_lints/src/...`, i.e. across a package boundary into another package's `src/`; §01's barrel rule says the barrel IS the public API and `src/` is private. [§01, cross-ref §02]
- Deviation: `packages/clean_architecture_core/lib/src/base/port.dart:1-9` vs `.../lib/src/base/repository.dart:1-9` - two near-identical base types with the SAME doc macro `{@template repository}` and duplicated wording; a fact stated twice is a D.R.Y. violation. [§01]
- Deviation: `README.md:8,32-54` and `packages/*/README.md` - repo prose restates tooling doctrine (melos activate/bootstrap/test steps) instead of referencing a single source; §01's D.R.Y. rule says repo docs reference a rule, never restate it. [§01]
- Deviation: `packages/architecture_lints/README.md` (608 lines, 72 code fences) and `packages/clean_architecture_lints_experimental/README.md` (identical 608 lines, 72 fences) - large code bodies live in prose docs; §01/§02 say code belongs in tests first and READMEs carry no rules. The two READMEs being byte-identical copies of each other is the drifting-double problem the bible names. [§01, cross-ref §02]
- Deviation: repo root - there is no `AGENTS.md` anywhere (`git ls-files` shows none), so the repo records no deviations and no local wiring; §01 D.R.Y. assigns exactly that job to `AGENTS.md`. [§01, cross-ref §04 error-style declaration]

### §02 — Toolchain & SDK
- Deviation: `packages/clean_architecture_kit/pubspec.yaml:12` - `sdk: ^3.9.0` sits below the §02 floor of `>=3.10.0` (all four other members use `^3.10.0`). [§02]
- Deviation: `analysis_options.yaml:14` - `public_member_api_docs: false` disables the one lint §02 names as the documentation gate (and §09 step 8 says to enable it). [§02, cross-ref §09]
- Deviation: `analysis_options.yaml:15-16` - `sort_constructors_first: false` and `lines_longer_than_80_chars: false` disable lints; §02 says never disable a rule in `analysis_options.yaml`. [§02]
- Deviation: `analysis_options.yaml` (root) - no `todo` -> `error` analyzer mapping exists anywhere in the repo, while 21 `TODO` comments are live in tracked `.dart` files; §02 makes a TODO a compile error. [§02, cross-ref §10]
- Deviation: `pubspec.yaml:16,19-33` - melos is pinned at `^7.3.0` (bible pins `^8.8.0`), the scripts use the melos-7 `run:` key (melos 8 requires `exec.command:`), and melos is configured as the workspace orchestrator; §02 makes melos OPTIONAL and scoped to single-package scripts. [§02, cross-ref §10]
- Deviation: `README.md:8,34-43` - the README presents melos as the monorepo's manager and tells contributors to `dart pub global activate melos`; §02 treats a melos-centric monorepo as a smell (config belongs in the root `pubspec.yaml` `melos:` key; no `melos.yaml` — that part is correct here). [§02]
- Deviation: `examples/clean_feature_first/analysis_options.yaml:1-8` - the example app's options do not include the root options at all: no `very_good_analysis` base, no `public_member_api_docs`, no `todo: error`, and `custom_lint: {debug: true}` is left on in a committed config. [§02, cross-ref §09 step 8]
- Deviation: repo-wide - no `dart_arch_test` dependency and no architecture/boundary test exists (`grep` for `arch_test`/`Collector.buildGraph` returns nothing); §02/§09 require the resolved import graph to be asserted in-test, not by melos and not by `import_rules`. [§02, cross-ref §09 step 7]
- Deviation: `packages/clean_architecture_lints_experimental/lib/**` (200 import sites, e.g. `lib/src/models/configs/naming_conventions_config.dart:8` via `part 'package:...'`) - cross-package `package:<other>/src/...` imports, the exact thing `implementation_imports` (already on via `package:lints/recommended`) flags; measured as 200 `implementation_imports` diagnostics in the sibling audit. [§02, cross-ref §01]
- Deviation: repo-wide formatting - a real read-only run, `dart format --output=none --set-exit-if-changed .`, reports **196 of 521 tracked `.dart` files** would be reformatted; §02 makes `dart format` the only formatter and requires a clean tree before committing. [§02]
- Deviation: 106 tracked `.dart` files lack a trailing newline (e.g. `examples/clean_feature_first/lib/core/entity/entity.dart`), the analyzer's `eol_at_end_of_file` family; §02's zero-diagnostics gate counts these. [§02]
- Deviation: `packages/clean_architecture_core/lib/src/base/{entity,params,port,repository, sources}.dart:1` and `lib/src/error/failure.dart:1` - six files open with a file-level `///` doc comment, the dangling-library-doc pattern §02 calls out (it attaches to the unnamed library and needs a `library;`, which the bible forbids adding for that purpose). There is no `library;` directive anywhere in the repo. [§02]
- Deviation: 172 files under a `lib/` directory contain no `///` comment at all, including every example-app source file and both use cases (`examples/.../domain/usecases/request_login.dart`, `request_logout.dart`); §02 requires a terse `///` on declarations and public members and makes use cases the documentation priority. [§02, cross-ref §01, §10]
- Deviation: `packages/architecture_lints/lib/architecture_lints.dart:1` (and `packages/clean_architecture_kit/lib/clean_architecture_kit.dart:1`) - the barrel has no doc comment at all; its first line is a plain `//` comment naming a *different, stale* file (`// lib/src/architecture_lints_plugin.dart`), so the package never declares its error style. [§02, cross-ref §04]
- Deviation: `packages/architecture_lints/analysis_options.yaml:1-8` and `packages/clean_architecture_lints_experimental/analysis_options.yaml:1-8` - committed `analyzer: exclude` blocks for `build/`, `android/`, `ios/`, `web/`, `windows/`, `macos/`, `linux/`; the sibling audit shows `flutter pub get` re-inserts them on every run, so the file is a tooling-managed artifact being hand-tracked. [§02]
- Deviation: `analysis_options.yaml:3-5` (root) - `analyzer: exclude: ["**/*.g.dart"]` is left over from a codegen era; no generated file exists in the tree. [§02, cross-ref §07]
- Deviation: `examples/clean_feature_first/pubspec.yaml:13-22` - seven dependencies are declared as `any` (`bloc`, `cached`, `dart_mappable`, `flutter_bloc`, `fpdart`, `injectable`, `custom_lint`); §02's stack is pinned. [§02, cross-ref §04, §07, §11]
- Deviation: `pubspec.yaml` / repo root - no CI workflow exists (`.github/` is absent), so §02's analyzer+test gate and §09 step 10's per-PR CI never run. [§02, cross-ref §09]
- Deviation: `analysis_options.yaml:8` + `packages/*/analysis_options.yaml` - analyzer config is spread over 5 files with a mix of include chains and local excludes rather than the single root-wired `analysis_options.yaml` §09 step 8 describes. [§02, cross-ref §09]
- Deviation: `packages/architecture_lints/lib/src/utils/naming_utils.dart:4` - declares `class NamingUtils2` (a second `NamingUtils` also exists at `packages/clean_architecture_kit/lib/src/utils_old/naming_utils.dart:7`); the numbered-class rename smell. [§02, cross-ref §03]

### §03 — Repository Topology & Package Layout
- Applicability: repo root layout - the member set is `architecture_lints`, `clean_architecture_core`, `clean_architecture_kit`, `clean_architecture_lints_experimental`, `examples/clean_feature_first`; there are no `*_domain`/`*_usecases`/`*_datasource_*` packages and no `apps/` ring. Neither sanctioned Topology A nor B describes a linter toolkit, so §03's package-naming and dependency matrix have no target. Flagged for review, not treated as a code fault. [§03]
- Deviation: 44 files under `packages/*/lib` and `examples/*/lib` declare more than one top-level type in a single file (e.g. `examples/.../lib/core/error/failures.dart` = `Failure` + `ServerFailure` + `CacheFailure`; `.../lib/core/usecase/usecase.dart` = 4 use-case bases; `.../presentation/managers/auth_bloc.dart` = `AuthBloc` + `AuthEvent` + `AuthState` + `AuthInitial`); §03 says one class per file, one file per class. [§03]
- Deviation: `packages/architecture_lints/lib/src/schema/constants/config_keys.dart:6-20` - one `class`-less file `part`s in 16 sibling `keys/*.dart` files; `part`/`part of` splintering is the opposite of the file-per-class layout §03 states. Same pattern across `packages/clean_architecture_lints_experimental/lib/src/models/**` (24 part/part-of files). [§03]
- Deviation: `packages/clean_architecture_lints_experimental/lib/src/models/configs/ naming_conventions_config.dart:8` - `part 'package:architecture_lints/src/models/rules/ naming_rule.dart';` uses a `package:` URI in a `part` directive (only relative URIs are legal), so this file can never compile. [§03, cross-ref §02]
- Deviation: `examples/clean_feature_first/lib/features/auth/data/models/ source.name.snakeCase_model.dart` - a file whose *name* is a mustache expression, committed into `lib/`; §03's layout admits Dart sources only, and the sibling audit shows it produces 62 analyzer errors (`uri_with_interpolation`, `undefined_identifier` on `source`). [§03, cross-ref §02, §07]
- Deviation: 12 `*.violations.dart` files committed under `examples/clean_feature_first/lib/**` (`model.violations.dart`, `repository.violations.dart`, `entity.violations.dart`, `usecase.violations.dart`, `port.violations.dart`, `login_screen.violations.dart`, `orphan_logic.violations.dart`, ...) - linter demo fixtures placed in the app's `lib/` tree; §02's code-placement rule makes tests the default home for example code. [§03, cross-ref §02]
- Deviation: `packages/storage.json` - a tracked 5-byte file of non-UTF8 bytes (`ff 0f 6f 7b 00`) sitting in the `packages/` directory; not a Dart source, not a package. [§03]
- Deviation: `.idea/**` (25 tracked paths incl. `caches/deviceStreaming.xml`) and the `*.iml` files (`melos_clean_architecture_workspace.iml`, `packages/*/melos_*.iml`, `packages/*/clean_ architecture_*.iml`) are committed; editor artifacts are not repo content in either topology. [§03]
- Deviation: `packages/architecture_lints/lib/src/lints/**` and `.../lib/src/schema/**`, `.../lib/src/engines/**` coexist with `.../lib/src/lints_old/**` and `.../lib/src/utils_old/**` (and the same split in `clean_architecture_kit` and `clean_architecture_lints_experimental/lib/src/utils_old/**`) - two parallel generations of the same code live in the source tree. [§03]
- Deviation: `packages/architecture_lints/test/src/lints_old/**` and `test/src/utils_old/**` - the test tree mirrors the OLD directory layout while `lib/` moved on, which is the direct cause of the 122 unresolvable import URIs and 79/115 unloadable test files in the sibling audit. [§03, cross-ref §06]
- Deviation: repo root `.gitignore` - it contains only `/.dart_tool/`, so nothing prevents `build/`, IDE folders or scratch artifacts from being committed again. [§03]

### §04 — The Functional Core (fpdart)
- Deviation: `packages/clean_architecture_core/pubspec.yaml:21` - `fpdart: ^1.0.0`; §04 pins `fpdart: ^1.2.0` everywhere. [§04]
- Deviation: `examples/clean_feature_first/pubspec.yaml:21` - `fpdart: any`. [§04]
- Deviation: repo-wide - `TaskEither` appears 0 times in 522 tracked `.dart` files; there is no typed-failure composition chain anywhere (no `flatMap`/`Do`/`.run()` seam), so §04's composition rules are unrealised rather than violated. Flagged as an applicability gap. [§04]
- Deviation: `examples/clean_feature_first/lib/features/auth/data/repositories/auth_repository.dart: 25-32` - hand-rolled `try { } on ServerException { } catch (e) { }` inside the repository adapter, with a bare `catch (e)` and a `ServerFailure(e.toString())` that stringifies the throwable; §04 allows only `TaskEither.tryCatch`/`Either.tryCatch` wrapping the third-party call at the adapter boundary, and hand-rolled try/catch in an adapter is named as a violation. [§04, cross-ref §10]
- Deviation: `examples/clean_feature_first/lib/features/auth/data/repositories/auth_repository.dart: 36-38` - `logout()` `throw UnimplementedError()` in a repository: a `throw` on the public API of a non-UI ring, which §04's exception rule forbids. [§04]
- Deviation: `examples/clean_feature_first/lib/features/auth/data/sources/auth_source.dart:12-16` - the datasource contract returns raw `Future<UserModel>` / `Future<void>`, not `Either`/`TaskEither` with a datasource failure hierarchy; §04/§05 require every datasource method to return a typed failure value with `tryCatch` living there. [§04, cross-ref §05]
- Deviation: `examples/clean_feature_first/lib/core/error/failures.dart:13-20` - the failure hierarchy is not closed (leaves are `class`, not `final`/`sealed`) and there is no per-layer split (`DatasourceFailure` family absent), so a `switch` over failures cannot be exhaustive and datasource failures are never mapped upward. [§04, cross-ref §05]
- Deviation: `packages/clean_architecture_core/lib/src/base/params.dart:1-9` - the core ships a `Params` base class, encoding the cargo-object pattern §04 bans ("no anonymous container types to define/serialize/pass across layers"). [§04, cross-ref §01]
- Deviation: `packages/clean_architecture_core/lib/src/base/usecase.dart:20-37` - `NullaryUsecase`/ `UnaryUsecase` are generic over `ParameterType` and take it positionally, which is how the example ends up passing a record container; §04 requires discrete named business inputs. [§04, cross-ref §01]
- Deviation: `packages/clean_architecture_core/lib/src/base/usecase.dart:11` - the base `Usecase` holds a `Repository` field and requires it in the constructor, forcing every use case to depend on a repository base type rather than the contract it actually needs. [§04]
- Deviation: error style is never declared - no barrel doc comment, no README section and no `AGENTS.md` states whether consumers receive `Future<Either<...>>` or exceptions (checked in all four barrels and three package READMEs). §04: "The violation is silence, never the choice." [§04, cross-ref §01, §10]
- Deviation: 66 `throw` occurrences across 20 files under `packages/*/lib` and `examples/*/lib` (e.g. `packages/architecture_lints/lib/src/engines/configuration/ config_loader.dart`, `.../engines/template/template_loader.dart`, `packages/clean_architecture_lints_experimental/lib/src/fixes/create_to_entity_method_fix.dart`) - these are linter-engine internals, not bulls-eye application code, so §04's exception rule does not clearly address them; combined with the previous item (no declared error style) they are flagged for review rather than asserted as violations. [§04]

### §05 — Persistence Doctrine
- Deviation: repo-wide - no `drift`, `drift_dev`, `build_runner`, `sembast` or `sqlite3` dependency exists in any pubspec, and no `IUnitOfWork` contract exists anywhere; §05's sanctioned sqlite3 ORM and file-store defaults are simply absent. [§05]
- Deviation: `examples/clean_feature_first/lib/features/auth/data/sources/auth_source.dart:7-16` - the datasource seam is declared in the DATA layer of a feature (`features/auth/data/sources/`), not in a domain package; §05 puts the contract in the domain package, which owns no implementation. Same for the repository contract usage: `AuthSource` is implemented only by `DefaultAuthSource`. [§05, cross-ref §03]
- Deviation: `examples/clean_feature_first/lib/features/auth/data/sources/default_auth_source.dart` - a single datasource adapter: no second adapter, no in-memory/second-store alternative, and no shared contract test suite run against every adapter ("a contract is only real when two implementations agree on it"). [§05]
- Deviation: `examples/clean_feature_first/lib/features/auth/data/repositories/auth_repository.dart: 13-33` - the repository neither exposes an optional `IUnitOfWork? uow` on writes nor maps datasource failures upward; §05 requires the uniform `uow` seam on write methods and per-adapter transaction handling (real wrap or honest sink). [§05]
- Deviation: `examples/clean_feature_first/lib/features/auth/data/models/user_model.dart:25-29` - hand-written `fromJson` exists, but there is no persistence layer, no wipe/clear path and no read-your-writes test; §05's transactional pitfalls (single-unit-of-work wipes, persistent-store regression tests) are unaddressed. [§05]

### §06 — Testing Doctrine
- Deviation: repo-wide stack - `shouldly` is not declared in any pubspec and appears 0 times in the tree, while `expect(` is used in 170 test files (1205 `test(` declarations); §06 makes shouldly the assertion library and bans mixing. [§06]
- Deviation: test names - only 4 occurrences of `Given `/`When `/`Then ` appear across 1205 tests; the suites name groups after classes (e.g. `packages/clean_architecture_kit/test/src/utils_old/layer_resolver_test.dart:65` `group('LayerResolver', ...)`), not the Given/When/Then idiom. [§06]
- Deviation: `packages/architecture_lints/test/**` and `packages/*/test/**` - `mocktail` is declared in three packages and `Mock`/`when(` appears 195 times in 21 files; §06 restricts mocks to the application-layer seam over interfaces the repo owns, which has no counterpart in a linter toolkit. Flagged as an applicability tension, not asserted as a violation. [§06]
- Deviation: `packages/clean_architecture_core/test/` (absent) - the core package declares `test: ^1.26.0` as a dev dependency but ships 0 test files; §06's domain layer is "unit tests, pure, no mocks" by default. [§06, cross-ref §09 step 6]
- Deviation: `examples/clean_feature_first/test/` (absent) - the Flutter app ships 0 test files; §06 requires widget tests at the app's `test/` that pump widgets and assert rendered state. [§06]
- Deviation: `examples/clean_feature_first/lib/features/auth/data/repositories/` - no shared contract test suite exists for `AuthPort`, and no adapter is exercised against one. [§06, cross-ref §01, §05]
- Deviation: `packages/architecture_lints/test/src/config/parsing/package_path_resolver_test.dart: 4-5`, `.../yaml_merger_test.dart:4-5`, `.../lints/boundaries/logic/component_logic_test.dart:4-5`, `.../lints/consistency/rules/orphan_file_rule_test.dart:4-5`, `.../lints/members/logic/member_logic_test.dart:4-5`, `.../lints/metadata/logic/annotation_logic_test.dart:4-5`, `.../lints/usages/logic/usage_logic_test.dart:4-5` - six test files whose whole body is `test('TODO: Implement tests for X', () { // TODO: Implement test });`; §06 wants real given/when/then assertions, and §02 makes a TODO a compile error. [§06, cross-ref §02]
- Deviation: `packages/architecture_lints/test/**` - 7 test files import `package:flutter_test/flutter_test.dart` although no pubspec in the workspace declares `flutter_test`; the suite cannot even be resolved (real failure in the sibling audit). [§06, cross-ref §02]
- Deviation: all three suites - `test/` paths still target the pre-refactor layout (`test/src/lints_old/**`, `test/src/utils_old/**`, and imports of deleted symbols such as `ArchComponent`, referenced by 53 tests and existing nowhere in the repo); the bible's testing doctrine assumes a suite that runs. [§06, cross-ref §03]

### §07 — Builders & Codegen
- Deviation: `examples/clean_feature_first/pubspec.yaml:17` - `dart_mappable: any` is a codegen mapper in the same family §07 rules out (freezed/json_serializable/riverpod_generator are banned and the rule is "we avoid codegen builders wherever we can"); no `dart_mappable_builder` and no generated `*.mapper.dart` exist, so the dependency is declared but unused. [§07]
- Deviation: `examples/clean_feature_first/templates/model_freezed.dart.mustache` - a committed template that emits `part '{{fileName}}.freezed.dart';`, `part '{{fileName}}.g.dart';` and `@freezed`; freezed is explicitly out per §07 (prefer `sealed` classes + records). [§07]
- Deviation: `examples/clean_feature_first/lib/features/auth/data/models/ source.name.snakeCase_model.dart:2-3` - the materialised output of that freezed template, committed to `lib/` with interpolated `part` names. [§07, cross-ref §03]
- Deviation: `packages/architecture_lints/lib/src/engines/template/**` (incl. `template_loader.dart`, `engines/generator/**`) and `packages/*/lib/src/fixes/**` - the packages ship their own mustache-driven code generator and quick-fix synthesizers; §07's test for whether a builder earns its place ("does the generated code encode a contract we'd otherwise hand-maintain and get wrong") is not addressed anywhere. The generation capability is the product here, so this is flagged as a doctrine-scope question, not a defect. [§07]
- Deviation: `packages/architecture_lints/lib/src/fixes/create_to_entity_method_fix.dart:99,111` and `packages/clean_architecture_lints_experimental/lib/src/fixes/create_to_entity_method_fix.dart: 135` - generated code embeds `TODO` text and `throw UnimplementedError(...)` into consumer sources it synthesises, seeding banned patterns (TODOs, throws) into user repos. [§07, cross-ref §02, §04]
- Deviation: `packages/architecture_lints/analysis_options.yaml:3-5` - the `**/*.g.dart` analyzer exclusion assumes a generated-code era that no longer exists (0 `*.g.dart` in the tree). [§07]

### §08 — Flutter: The Outer Ring
- Deviation: `examples/clean_feature_first/pubspec.yaml:13,18` + `lib/features/auth/presentation/managers/auth_bloc.dart:3,9` (`import 'package:bloc/bloc.dart'`, `class AuthBloc extends Cubit<void>`) - state is bloc/Cubit, not `flutter_riverpod` plain providers; `riverpod` appears 0 times in the repo. §08/§11 settle Riverpod with no generator. [§08, cross-ref §11]
- Deviation: `examples/clean_feature_first/lib/core/error/failures.dart:6,13-20` - domain failures carry display strings (`final String message`, `'Server Failure'`), and the repository returns them directly; §08 says the error-message text is derived from the failure type at the boundary and domain failures never carry display strings. [§08, cross-ref §01, §04]
- Deviation: `examples/clean_feature_first/lib/features/auth/presentation/managers/ login_controller.dart:4` - `import 'package:flutter/material.dart'` in a presentation "manager" that holds business logic (the file itself is annotated `//! <-- WARNING`); §08 keeps widgets dumb and business rules out of the widget/controller layer. [§08]
- Deviation: `examples/clean_feature_first/lib/core/utils/service_locator.dart:7-12` - a global `getIt` locator re-exported to every consumer, i.e. a widget-reachable path that skips the use-case seam; the sibling audit shows the repo's own `arch_usage_global_access` rule is configured to flag exactly this. [§08, cross-ref §01, §11]
- Deviation: `examples/clean_feature_first/lib/features/auth/presentation/managers/ login_controller.dart:11-15` - `LoginController extends ChangeNotifier` with a `login()` that does nothing; there is no provider wrapping a use case, so §08's "use cases are exposed as providers; widgets reach use cases through providers" has no implementation in the example. [§08]
- Deviation: `examples/clean_feature_first/lib/features/auth/presentation/pages/home_page.dart: 13-25` - `SomePage`/`_SomePageState` is a second widget added to a page file that renders `Placeholder()`; §08 asks for small `build` methods and extracted `StatelessWidget`s, and the sibling audit flags this file with `arch_orphan_file`. [§08]
- Deviation: `examples/clean_feature_first/lib/features/auth/presentation/widgets/user_avatar.dart` - a presentation widget annotated as depending on a page ("Widgets should not depend on Pages"), i.e. the example ships an inverted widget dependency. [§08]
- Deviation: `examples/clean_feature_first/analysis_options.yaml:17-18` - `custom_lint: debug: true` committed in the app config keeps verbose linter logging on for every analysis; not a UI-ring rule but part of the same boundary config the app presents as its lint gate. [§08, cross-ref §02]

### §09 — Bootstrap Checklist (new project)
- Deviation: repo root - checkbox 10 has no artifact: there is no CI configuration at all (`.github/` absent), so `dart analyze --fatal-infos --fatal-warnings` + `dart test` never run on a PR. [§09]
- Deviation: repo root - checkbox 7 has no artifact: the `dart_arch_test` boundary test (import-graph direction + cycle freedom) does not exist. [§09, cross-ref §02]
- Deviation: `analysis_options.yaml:14-16` - checkbox 8's `public_member_api_docs` is set to `false` and `todo: error` is absent. [§09, cross-ref §02]
- Deviation: workspace member set (root `pubspec.yaml:8-12`) - checkbox 3's `*_domain`, `*_usecases` and two `*_datasource_*` packages were never created; `resolution: workspace` in every member (checkbox 4) and the explicit `workspace:` list (no globs) ARE satisfied. [§09, cross-ref §03]
- Deviation: `examples/clean_feature_first/lib/features/auth/domain/usecases/request_login.dart` - checkbox 9's deliverable (first use case with terse `///` docs, business-param `call()`, named params, plus a mocktail test covering both `Either` sides) exists only partially: the use case has no doc comment, takes a cargo record positionally, calls the port with positional args, and has no test at all. [§09, cross-ref §04, §06]
- Deviation: `packages/architecture_lints/analysis_options.yaml:1-8` - checkbox 8's root-wired single `analysis_options.yaml` is contradicted by per-package option files carrying their own `analyzer:` blocks. [§09, cross-ref §02]

### §10 — Review Checklist
- Deviation: repo root - "Does `dart analyze --fatal-infos --fatal-warnings` report zero diagnostics and `dart test` green?" -> **no**: real runs give `dart analyze` exit 3 with 3825 issues (2452 error / 235 warning / 1138 info), `dart analyze --fatal-infos` exit 3, and all three suites exit 1 (`architecture_lints` +242 -115 with 79/115 files failing to load; `clean_architecture_kit` +0 -9 with 9/9 failing to load; `clean_architecture_lints_experimental` +24 -54 with 54/55 failing to load). [§10, cross-ref §02]
- Deviation: `examples/clean_feature_first` - the repo's own enforcement disagrees with itself: `dart run custom_lint` prints "No issues found!" (exit 0) while `dart analyze` emits 5 real `arch_*` findings in the same tree and the analyzer logs an `analysis.setAnalysisRoots` plugin crash; a review checklist cannot be run against a gate that reports two different answers. [§10, cross-ref §02]
- Deviation: repo-wide - "no `TODO`/`FIXME` comments anywhere?" -> no: 21 TODO comments are live in tracked `.dart` files (6 of them entire stub test bodies). [§10, cross-ref §02]
- Deviation: repo-wide - "any `// ignore:` per-line with a reason; no `ignore_for_file`?" -> partially: 0 `ignore_for_file` (good) and 30 `// ignore:` lines, but several are blanket rule silences on a declaration (e.g. `examples/.../presentation/managers/auth_bloc.dart:8` `// ignore: arch_type_missing_base`) rather than an explained per-line exception. [§10]
- Deviation: repo-wide - "`melos.yaml` anywhere?" -> correctly absent; but melos is pinned and configured as the workspace orchestrator in the root `pubspec.yaml:16-33`, which §10 asks a review to challenge. [§10, cross-ref §02]
- Deviation: `packages/architecture_lints/pubspec.yaml` - `mocktail`/`test` declared while the README (`packages/architecture_lints/README.md`) recommends `fpdart: ^1.1.0`, the code pins `^1.0.0`, the example uses `any`, and the bible pins `^1.2.0`: four different statements of one fact, the D.R.Y. failure §10's last bullet asks a reviewer to catch. [§10, cross-ref §01, §04]

### §11 — Decisions — Change Record
- Deviation: `examples/clean_feature_first/lib/features/auth/domain/usecases/request_login.dart:7, 11` and `.../data/repositories/auth_repository.dart:10,12` - `import 'package:injectable/ injectable.dart'` with `@Injectable()` / `@LazySingleton(as: AuthPort)`; §11 settles manual constructor injection (no get_it/injectable) and §07 bans the generator that would consume these annotations. [§11, cross-ref §07]
- Deviation: `examples/clean_feature_first/lib/core/utils/service_locator.dart:1-9` - `get_it` is the service locator (`GetIt.instance`, re-exported), directly contradicting the settled decision that composition happens at the app's composition root. [§11, cross-ref §01, §08]
- Deviation: `examples/clean_feature_first/pubspec.yaml:13,18` + `lib/features/auth/presentation/managers/**` - bloc/Cubit is the state solution, not the settled Riverpod-with-plain-providers decision. [§11, cross-ref §08]
- Deviation: `examples/clean_feature_first/lib/main.dart:1-20` - there is no composition root at all: `main()` runs `MyApp` directly, no `ProviderScope`, no adapter wiring, and `configureDependencies()` in the service locator is an empty stub. [§11, cross-ref §03, §08]
- Deviation: `examples/clean_feature_first/lib/` - `go_router` (the settled navigation decision) is absent; navigation is a single hard-coded `home: HomePage()`. Not a violation of a banned practice, but the example demonstrates none of the §11 navigation doctrine. [§11]
- Deviation: repo root - the repo keeps no decisions record of its own (no `AGENTS.md`, no decisions doc), so nothing states where this toolkit deviates from the bible or why. [§11, cross-ref §01]

### §12 — Sources of Truth
- Deviation: `README.md:12-30` and `packages/*/README.md` - no repo document points at the bible or at the `dart-clean-architecture` skill; the toolkit presents its own README as the source of truth, which is the drift §12 exists to prevent. [§12, cross-ref §01]
- Deviation: `packages/architecture_lints/README.md` and `packages/clean_architecture_lints_experimental/README.md` (both 608 lines, byte-identical, 72 code fences each) - neither names a canonical source and both duplicate the other; §12/§01 treat a second copy of a truth as suspect. [§12, cross-ref §01]
- Deviation: `examples/clean_feature_first/architecture.yaml:3-4` - the config file's own `#include` lines point at `../../architecture_standards/clean_feature_first_recommended.yaml` and `package:acme_architecture/presets/clean_architecture.yaml`; neither path exists in the repo or in any declared dependency, so the example's stated source of architectural truth is missing (both lines are commented out, and the linter has no include mechanism to resolve them). [§12]

