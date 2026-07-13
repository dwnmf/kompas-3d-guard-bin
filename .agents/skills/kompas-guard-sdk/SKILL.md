---
name: kompas-guard-sdk
description: Find, inspect, ground, and verify KOMPAS-3D API usage through the local in-process SDK. Use whenever a task requires KOMPAS API members, acquisition paths, signatures, get/set distinctions, code verification, or an explicitly requested live KOMPAS run. Never guess API names from memory.
---

# KOMPAS Guard SDK

Use the direct Python SDK. Do not start services or call localhost endpoints.

Install the protected Windows release with `py -3.12 -m pip install -U kompas-3d-guard`.
The first `KompasGuard()` downloads and verifies the pinned public Core model;
subsequent calls reuse the local cache. Do not request or embed Hugging Face credentials.

## Required workflow

1. Collect all natural-language API questions and call one `batch(..., k=5)`.
2. Read top 1–3 compact results for every query.
3. Call `batch.inspect(...)` for ambiguous owners, get/set pairs, close scores, or partial paths.
4. Use `context(task)` when generation needs combined compact grounding.
5. For standards questions, use `gost_current`, then `gost_text_search` or `gost_section`; retain source SHA in the answer.
6. Call `verify(code)` before presenting executable KOMPAS automation code.
7. Call `run(...)` only when the user requests live execution; it controls desktop KOMPAS through an isolated worker.

```python
from kompas_guard import KompasGuard

kg = KompasGuard()
batch = kg.batch(["получить радиус окружности", "сохранить документ детали"], k=5)
print(batch.render("compact"))
details = batch.inspect_many(["Q1.R1", "Q1.R2"])
```

Treat `path|partial:...` as unresolved. Never invent missing acquisition steps.
If a symbol has both get and set operations, choose using the requested action and confirm through `related`.

`verify()` is conservative: unsupported or unknown Python shapes are not proof of correctness. A successful live run is the strongest evidence, followed by compiler verification when available, then graph verification.
