# Справочник KOMPAS Guard SDK 0.6.2

## Содержание

1. Python API
2. Result objects
3. CLI
4. Окружение и кэш
5. Проверяемый API7 solid workflow

## Python API

### Создание SDK

```python
KompasGuard(
    bundle_path=None,
    graph_path=None,
    reranker_path=None,
    retriever=None,
    status=None,
)
```

`bundle_path` выбирает локальный immutable Core. `status(message)` получает
состояние первой загрузки и проверки. `retriever` предназначен для тестов.

### Retrieval

```python
kg.search(query, k=5) -> list[SearchResult]
kg.batch(queries, k=5) -> SearchBatch
kg.context(task, k=5) -> GroundedContext
kg.inspect(identifier, detail="compact") -> dict
kg.graph_coverage_audit() -> dict
```

`identifier` принимает document id, stable ref или однозначный exact symbol.
При get/set ambiguity передавать stable ref результата.

### Constants

```python
kg.enum_values(enum, contains=None) -> EnumResult
kg.constant(identifier) -> ConstantValue
kg.constants(query, enum=None, k=5) -> ConstantBatch
kg.app_const(name) -> AppConstantResult
kg.metadata_audit() -> dict
```

`constant` принимает fully-qualified `Enum.Member`; короткий member разрешается
только при одном совпадении. `app_const` возвращает явно curated domain groups,
а не typelib enum.

### Paths

```python
kg.path(to_type, from_type="IApplication") -> PathResult
kg.coclass(interface) -> CoclassResult
```

`PathResult.found`, `.steps` и `.alternatives` программно доступны. Каждый
`PathStep` содержит kind, source/target type, member, verified, source и
provenance.

### Verification

```python
kg.verify(code, task="", backend=None) -> VerificationResult
kg.verify_file(path, task="", backend=None) -> VerificationResult
```

Backend: `auto`, `compiler`, `graph`. Основной человекочитаемый результат —
`VerificationResult.feedback`, не JSON.

### Runner

```python
kg.run(code, setup=None, timeout_s=None, check=None, cleanup="none") -> RunResult
kg.run_file(path, setup=None, timeout_s=None, check=None, cleanup="none") -> RunResult
kg.classify(result, task="", code="") -> RuntimeClassification
```

Допустимые cleanup policies: `none`, `created_unsaved`, `created`.

### GOST

```python
kg.gost_current(identifier)
kg.gost_search(query, k=3)
kg.gost_text_search(query, gost_id=None, k=8, current_only=True)
kg.gost_section(gost_id, heading, k=12)
```

## Result objects

Основные immutable objects поддерживают `render("compact")` и `to_dict()` либо
`render("json")`. Для общения с человеком использовать compact. JSON-проекция
нужна только явной программной интеграции.

- `SearchBatch`: queries, ranks, refs, scores, signatures, paths.
- `GroundedContext`: batch, recipes, standards, warnings, related enums, paths.
- `ConstantValue`: enum, name, numeric value, aliases, provenance, conflict, ref.
- `EnumResult`: enum header и filtered values.
- `ConstantBatch`: результаты RU/EN поиска.
- `PathResult`: основной путь и ranked alternatives.
- `CoclassResult`: coclasses и cointerfaces.
- `VerificationResult`: ok, backend, stages, diagnostics, warnings, candidates.
- `RunResult`: ok, check_ok, successful, state, cleanup и bounded diagnostics.

## CLI

Все команды принимают глобальные `--bundle`, `--graph`, `--reranker` перед
именем команды.

```powershell
kompas-guard search "создать окружность" "save part document" -k 5
kompas-guard context "создать деталь и выдавить эскиз" -k 5
kompas-guard inspect IPart7.DefaultObject --detail compact
kompas-guard enum-values ksObj3dTypeEnum --contains plane
kompas-guard constant ksObj3dTypeEnum.o3d_planeXOY
kompas-guard constants "базовая плоскость XOY" --enum ksObj3dTypeEnum -k 5
kompas-guard app-const KompasStampCellEnum
kompas-guard path IModelContainer --from-type IPart7
kompas-guard coclass IPart7
kompas-guard metadata-audit
kompas-guard verify candidate.py --task "создать фланец" --backend auto
kompas-guard run candidate.py --check check.py --cleanup created --result --verbose
kompas-guard gost-current 2.104-2006
kompas-guard gost-search "основная надпись" -k 3
kompas-guard gost-text-search "форма 1" --gost-id "Р 2.104-2023" -k 8
kompas-guard gost-section "Р 2.104-2023" "Основные надписи" -k 12
```

CLI выводит compact text для retrieval, constants, paths, verify и run.

## Окружение и кэш

- `KOMPAS_GUARD_BUNDLE`: локальный Core snapshot.
- `KOMPAS_GUARD_GRAPH`: отдельная read-only graph projection.
- `KOMPAS_GUARD_CACHE`: каталог Hugging Face cache.
- `KOMPAS_GUARD_OFFLINE=1`: запрет сетевой загрузки, требовать кэш.
- `KOMPAS_GUARD_QUIET=1`: скрыть status в stderr.
- `KOMPAS_GUARD_RERANKER=top3`: включить optional quality pack.
- `KOMPAS_VERIFY_BACKEND`: `auto`, `compiler` или `graph`.
- `KOMPAS_INTEROP_DIR`: локальная metadata directory для compiler proof.
- `KOMPAS_CSC`: явный путь к C# compiler.

Без установленного КОМПАС работают retrieval, constants, paths, GOST и graph
verify. Live run требует Windows, зарегистрированный KOMPAS API7 и pywin32.
Authoritative constants в Core не требуют COM registration, `gen_py` или csc.

Поддерживаются стандартные Windows x64 builds CPython 3.10, 3.11, 3.12, 3.13
и 3.14. Для каждой minor-версии устанавливается отдельный ABI-wheel со своим
встроенным worker. Free-threaded (`3.13t/3.14t`) и 32-bit Python не заявлены.

## Проверяемый API7 solid workflow

Для каждой задачи заново получить exact symbols и constants. Общая цепочка:

1. `IDocuments.Add` с `DocumentTypeEnum.ksDocumentPart`.
2. cast документа в `IKompasDocument3D` по verified path.
3. `IKompasDocument3D.TopPart`, затем verified cast `IPart7 → IModelContainer`.
4. `IPart7.DefaultObject` с `ksObj3dTypeEnum.o3d_planeXOY`.
5. `IModelContainer.Sketchs`, `ISketchs.Add`, plane, update.
6. `ISketch.BeginEdit`; получить active view через manager/views; cast view в
   `IDrawingContainer`; получить нужную geometry collection.
7. Завершить edit, получить feature collection, задать параметры операции и
   вызвать update/rebuild.
8. Выполнить read-only check количества объектов/габарита.
9. Сохранить через найденный `SaveAs` exact symbol.

Этот список описывает acquisition shape, но не заменяет `inspect`, enum lookup,
`path` и `verify` для конкретного candidate.
