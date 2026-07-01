---
name: kompas-guard-sdk
description: >-
  Use when writing/fixing Python COM code against KOMPAS-3D SDK, or when
  grounded facts are needed for KOMPAS API or ГОСТ/ЕСКД/СПДС standards. Drives
  KOMPAS-3D_GUARD via `from kompas_guard import Client`: context before
  generation, compiler/static verification after generation, optional live
  runner/classifier repair, and GOST freshness checks/current replacements.
---

# KOMPAS-3D_GUARD SDK skill

KOMPAS-3D_GUARD is a local grounding, verification and live-run layer. It does
not generate code; you write normal Python `def run(app): ...`, while the tool
supplies real KOMPAS API facts, checks hallucinated calls, runs against live
KOMPAS when available, and now grounds/checks ГОСТ/ЕСКД/СПДС freshness.

Core engines:

1. **Graph/search before generation** — real types, members, signatures, enums,
   interface paths/casts, recipes, and GOST facts/current standards.
2. **Compiler/static verifier after generation** — missing members/types, wrong
   signatures, property-vs-method, invalid casts/enums, sandbox builtin issues,
   plus GOST obsolete-reference warnings.
3. **Live runner** — Windows COM truth for state/null/HRESULT/runtime failures;
   classifier returns repair feedback.

## Start / health

Source/dev install:

```powershell
kompas-guard up       # context + runner; embedding optional/off by default
kompas-guard status   # context, runner, optional embed, KOMPAS install path
kompas-guard doctor   # diagnose SDK/service wiring
```

Windows pip wheel release (v0.5.1+): install the Windows x64 wheel and use the
normal CLI/SDK entry points. The wheel contains a thin Python SDK plus an
embedded compiled Nuitka runtime under `kompas_guard/_app`.

```cmd
pip install kompas_3d_guard-0.5.1-py3-none-win_amd64.whl
kompas-guard up
kompas-guard status
kompas-guard down
```

Python compatibility for this wheel: CPython 3.10–3.14 on Windows x64. It is not
for Linux/macOS, 32-bit Windows, or Windows ARM64. `Client()` auto-discovers the
embedded runtime; no `app_dir` is needed after wheel install.

Windows portable ZIP release (v0.5+): unzip the ZIP and run from the bundle root:

```cmd
kompas-guard.exe up
kompas-guard.exe status
kompas-guard.exe down
```

The portable ZIP and embedded wheel use a compiled runtime: root
`kompas-guard.exe` or the SDK wrapper dispatches to
`kompas-guard-runtime\kompas-guard-runtime.exe` (Nuitka). Do **not** call the
runtime exe directly unless debugging; use `kompas-guard.exe` / `.cmd`, the
installed `kompas-guard` console script, or SDK `Client(...)`.

`up` resolves app dir from embedded wheel runtime, `--app-dir`, `KOMPAS_APP_DIR`,
config, executable bundle root or CWD. Grounding/verify Tier0/1 work without
live KOMPAS; live `run` needs Windows + registered KOMPAS COM. Optional
dense/llama embedding is only for A/B: `kompas-guard up --with-embed`.

## Candidate contract

Write a file exposing exactly:

```python
def run(app):
    # app is live KOMPAS IApplication COM object.
    doc = app.ActiveDocument  # can be None unless setup guarantees state
    return doc.Name           # return a real value; don't print-only
```

Runner passes `result = run(app)` into checks. Declare setup instead of assuming
state.

## Core loop

```python
from kompas_guard import Client

kg = Client()                         # embedded wheel/config/env/services.json/localhost
# After installing the v0.5.1+ Windows wheel, Client() auto-discovers the
# embedded compiled runtime in site-packages. For an external portable ZIP,
# prefer:
# kg = Client(app_dir=r"C:\path\to\KOMPAS-3D_GUARD-portable-0.5.0-win-x64")
# or set KOMPAS_APP_DIR to that bundle root before constructing Client.
task = "Get active document name."

# 1) Ground first: API + GOST facts before writing code.
ctx = kg.context(task)
print(ctx.feedback)

# 2) Write candidate.py with def run(app): ...

# 3) Verify structure/freshness before live run.
v = kg.verify_file("candidate.py")
if v.ok is False:
    print(v.feedback)                 # fix diagnostics, then re-verify

# 4) Run only after verify has no hard blockers.
r = kg.run_file("candidate.py", setup=["ensure_3d_doc"], risk="read")
if not r.ok:
    c = kg.classify(r, task=task, code_path="candidate.py", context=ctx.context)
    print(c.failure_class, c.repair_prompt)
    # patch from repair_prompt, then verify -> run again
```

`kg.check_file(task, "candidate.py", live=True)` executes context→verify→run→
classify for one already-written candidate. It is an evaluator, not a solver.

## API grounding primitives

Use before coding or when answering “is this real?”:

```python
kg.manifest()                              # served type libs, SHA, counts
kg.symbol("IApplication.ActiveDocument")   # real signature; .valid/.suggestions
kg.symbol("IPart7.NewEntity")              # member validation
kg.path("IPart7", from_type="IApplication")# transition/cast path
kg.search("active document", k=8)          # hybrid retrieval by default
kg.members_of("IPart7", include_via=True)  # real members/signatures
kg.owners_of_member("Sketchs", exclude_collections=True)
kg.coclass("Part7")                        # QI/co-implemented interfaces
kg.enum_values("ksObj3dTypeEnum", contains="plane")
kg.app_const("KompasStampCellEnum")        # curated app/domain constants
```

`symbol(...).valid is False` means hallucinated; use `.suggestions`/`search`.
Use `strict=True` only for old callers that need exceptions. `context()` also
returns `.paths_structured`, `.recipes`, `.primary_type`, `.related_enums`; use
structured fields instead of parsing text when possible.

## GOST/ЕСКД/СПДС grounding and freshness

The model must **not** rely on memory for standards. `kg.context(task)` injects a
`GOST grounding` block when the task mentions ГОСТ/ЕСКД/СПДС/drawing terms
(`штамп`, `основная надпись`, `чертёж`, `спецификация`, `шероховатость`, `УГО`,
`СПДС`, etc.). It uses processed local data in `data/gost/` (override:
`KOMPAS_GOST_KB`) with current/obsolete catalogs and ref/replaces graph.

Direct SDK calls:

```python
kg.gost("2.104")                 # lookup row + status/current
kg.gost_current("2.104-2006")    # -> Р 2.104-2023
kg.gost_search("штамп", k=5)     # current standards, e.g. Р 2.104-2023
kg.gost_refs("Р 2.104-2023")     # formal normative references
kg.gost_text_search("форма 1", gost_id="Р 2.104-2023")  # cited text chunks
kg.gost_section("Р 2.104-2023", "Основные надписи")     # section chunks
kg.gost_report()                 # freshness report markdown
```

Freshness rule: if a standard is `superseded`, use its current replacement
unless the user explicitly asks for historical compatibility. If generated
code/comments mention obsolete IDs, `verify_file()` emits non-blocking warnings,
e.g. `ГОСТ 2.104-2006` → `ГОСТ Р 2.104-2023`. For exact requirements, do not
answer from memory: call `gost_text_search`/`gost_section` and cite returned
fragments (`heading`, `fragment`, `source_file`, `source_sha256`). Withdrawn
fastener standards may include practical ISO analogs.

Important examples:

- stamp/main title block: **ГОСТ Р 2.104-2023**; old `2.104-2006` obsolete since
  2024-03-01.
- text docs/specs: **Р 2.105-2019**, **Р 2.106-2019**.
- main drawing requirements: **Р 2.109-2023**.
- changes: **Р 2.503-2023**.
- roughness: `2789-73` / `Р ИСО 1302-2002`.

## Method reference

All results have `.ok`, `.passed` compatibility, `.request_id`, `.status`,
`.feedback`, `.raw`.

| Method | Purpose / key fields |
|---|---|
| `health()` / `doctor()` | service, compiler, runner, retrieval health |
| `version_info()` / `manifest()` | SDK/API/KB version, served type libs/counts |
| `symbol`, `members_of`, `owners_of_member`, `coclass`, `enum_values`, `app_const` | API graph facts |
| `path(to, from_type=...)` | interface path/casts: `.found`, `.steps`, `.text` |
| `search(q,k,backend=...)` | API-symbol hits |
| `context(task, ...)` | grounding context: API + GOST, paths, recipes, enums |
| `verify` / `verify_file` | compiler/static/GOST warnings: `.diagnostics`, `.warnings`, `.feedback` |
| `setup(steps)` | runner preconditions |
| `run` / `run_file` | live execution: `.ok`, `.status`, `.result_repr`, exception info |
| `classify(run, task=..., code_path=..., context=...)` | failure class + repair prompt |
| `check_file(task,path,live=True)` | full loop result with sub-results |
| `gost`, `gost_current`, `gost_search`, `gost_refs`, `gost_text_search`, `gost_section`, `gost_report` | GOST KB |
| `reset(close_documents=True)` | reset KOMPAS state |

## Setup, risk, failure classes

Runner setup steps: `kompas_running`, `close_all`, `ensure_3d_doc`,
`doc_3d_opened`, `ensure_part_doc`, `ensure_drawing_doc`,
`ensure_assembly_doc`. Most null failures are missing setup.

Risk: `read`, `query`, `create`, `modify`, `safe_read`, `mutating`. Declare it
honestly. Runtime classes: `missing_setup`, `null_object`, `attribute_error`,
`com_error`, `timeout`, `result_failed`, `unknown`.

## Rules of thumb

- **Ground before generating**: `context()`/`search()`/GOST calls first.
- **Verify before run**: only `run_file()` once verify has no hard blockers.
- **Use files** for verify/run (`verify_file`, `run_file`) to avoid huge JSON.
- **Declare setup** (3D task → `ensure_3d_doc`; drawing task →
  `ensure_drawing_doc`).
- **Classify runtime failures**, patch from `repair_prompt`, then verify→run.
- **Use current standards** from GOST grounding; obsolete ГОСТs are warning-level
  unless strict policy is later enabled.
- Verify backend per request must be `auto`; server chooses
  `KOMPAS_VERIFY_BACKEND=auto|interop|stubs|graph`.
- Retrieval default is zero-embedding hybrid BM25+graph+compiler rerank. Use
  `bm25` for fastest lexical baseline; `embed`/`llamacpp` only for explicit A/B.
- Raw submitted code is not logged; hashes/summaries only.

## Config / wheel/portable SDK invocation

Resolution order: explicit `Client(...)` args → env → embedded wheel runtime →
`services.json` → user config → Windows Credential Manager secrets → localhost
defaults.

```python
# v0.5.1+ pip-installed Windows wheel: auto-discovers embedded compiled runtime.
kg = Client(autostart=True)

# Already-running remote/local services:
kg = Client(hosted_url="http://127.0.0.1:8000",
            runner_url="http://127.0.0.1:8765",
            api_key=None, runner_token=None)

# External portable ZIP:
kg = Client(app_dir=r"C:\Tools\KOMPAS-3D_GUARD-portable-0.5.0-win-x64",
            autostart=True)
```

For portable autostart, `app_dir`/`KOMPAS_APP_DIR` must point at the ZIP root
(the folder containing `kompas-guard.exe`, `data/`, `portable_manifest.json`,
`kompas-guard-runtime/`). For the v0.5.1+ embedded wheel, `Client()` points at
`site-packages/kompas_guard/_app` automatically. The SDK starts compiled services
via the bundled runtime and refreshes URLs from `.tmp/run/services.json`.

Key env: `KOMPAS_CONTEXT_URL`, `KOMPAS_RUNNER_URL`, `KOMPAS_API_KEY`,
`KOMPAS_RUNNER_TOKEN`, `KOMPAS_APP_DIR`, `KOMPAS_RETRIEVAL_BACKEND`,
`KOMPAS_GOST_KB`, `KOMPAS_SDK_CACHE`, `KOMPAS_COMPILER_RERANK=1`,
`KOMPAS_HYBRID_DISABLE_DENSE=1`, `KOMPAS_MULTI_QUERY_MAX=3`.

KOMPAS installation autodetect (v0.5+): `KOMPAS_BIN` override → ASCON registry
(newest KOMPAS-3D) → `KOMPAS.Application.7` ProgID/CLSID → Program Files glob →
legacy fallback. `kompas-guard status` and admin state show detected `bin_dir`.
Set `KOMPAS_BIN` only if autodetect misses a non-standard install.

Interop compiler verification: override with `KOMPAS_INTEROP_DIR` if interop DLLs
are not in bundled `data/interop` or `<KOMPAS root>\Libs\PolynomLib\Bin\Client`.
If Windows verification falls back to `stubs`/`graph`, `/v1/verify-code` returns
a non-blocking `INTEROP_NOT_FOUND` warning; treat it as weaker verification.

## CLI equivalents

```powershell
kompas-guard doctor
kompas-guard status
kompas-guard up / down / restart
# Portable ZIP: run the same commands via .\kompas-guard.exe from bundle root
kompas-guard context --task "..."
kompas-guard verify --file candidate.py --json-out
kompas-guard run --file candidate.py --setup ensure_3d_doc
kompas-guard classify --file runner-result.json --task "..." --code-file candidate.py
kompas-guard check --task-file task.txt --file candidate.py --setup ensure_3d_doc --live
kompas-guard gost 2.104
kompas-guard gost-current 2.104-2006
kompas-guard gost-search "штамп основная надпись"
kompas-guard gost-refs "Р 2.104-2023"
kompas-guard gost-text-search "форма 1" --gost-id "Р 2.104-2023"
kompas-guard gost-section "Р 2.104-2023" --heading "Основные надписи"
kompas-guard gost-report
kompas-guard sdk --section agent-flow
```

CLI and `kompas-guard-mcp` are thin wrappers over the SDK: same config,
auth, semantics, request IDs, and logs.
