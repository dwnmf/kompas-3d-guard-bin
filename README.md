# KOMPAS-3D_GUARD Binary Distribution

Binary-only public distribution for **KOMPAS-3D_GUARD**.

This repository contains release metadata, agent skill instructions, and GitHub
Release assets. It does **not** contain the private source packages that implement
the verifier, context service, graph/GOST engines, or live runner.

## Install

Download the wheel from the latest GitHub Release and install it:

```bash
pip install kompas_3d_guard-0.5.1-py3-none-win_amd64.whl
```

Then use either the CLI:

```bash
kompas-guard up
kompas-guard status
kompas-guard context --task "Проверить доступность IApplication.ActiveDocument"
kompas-guard down
```

or the Python SDK:

```python
from kompas_guard import Client

kg = Client(autostart=True)
print(kg.health().ok)
```

The SDK auto-discovers the embedded compiled runtime installed into:

```text
site-packages/kompas_guard/_app
```

## Supported platform

The current wheel is:

```text
kompas_3d_guard-0.5.1-py3-none-win_amd64.whl
```

Compatibility:

- Windows x64 only
- CPython 3.10, 3.11, 3.12, 3.13, 3.14
- Requires local KOMPAS-3D COM installation for live runner operations

Not supported by this wheel:

- Linux/macOS
- 32-bit Windows
- Windows ARM64

## What is inside the wheel

The wheel contains:

- a thin Python SDK/CLI wrapper: `kompas_guard`
- an embedded source-free Nuitka runtime under `kompas_guard/_app`
- processed runtime data needed by the local service

The wheel does not ship private Python source packages such as:

```text
serve/
runner_service/
scripts/
```

## Agent skill

The pi/agent skill is included at:

```text
.agents/skills/kompas-guard-sdk/SKILL.md
```

Use it when writing or verifying Python COM code against the KOMPAS-3D SDK, or
when grounding answers in KOMPAS API / ГОСТ / ЕСКД / СПДС facts.

## Release integrity

Each release includes a `.sha256` file next to the wheel. Verify with:

```bash
certutil -hashfile kompas_3d_guard-0.5.1-py3-none-win_amd64.whl SHA256
```

Expected SHA256 for v0.5.1:

```text
e368b1d9b04dcba65c5bbe6fb2d91a372fe6e84ad240737ddf1738f26cd9fa61
```

## Upstream

Source development happens in the private/source repository. This repository is
intended for binary releases and agent-consumable documentation only.
