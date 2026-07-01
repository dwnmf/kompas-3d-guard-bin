# KOMPAS-3D_GUARD

Публичная бинарная сборка **KOMPAS-3D_GUARD** для пользователей и агентов,
которые работают с КОМПАС-3D через Python SDK/COM.

Репозиторий содержит только:

- готовый wheel в разделе **Releases**;
- краткую пользовательскую документацию;
- skill для pi/agent: `.agents/skills/kompas-guard-sdk/SKILL.md`.

Исходный код внутренних сервисов, verifier, runner, GOST/API engines и private
Python-пакеты здесь не публикуются.

## Что это даёт

KOMPAS-3D_GUARD помогает агенту или Python-скрипту безопаснее работать с
КОМПАС-3D SDK:

- подсказывает реальные интерфейсы, методы, свойства и enum'ы KOMPАС API;
- проверяет Python COM-код до запуска;
- запускает код в живом КОМПАС-3D через локальный runner;
- классифицирует ошибки COM/HRESULT/null-state;
- даёт grounding по ГОСТ / ЕСКД / СПДС, включая актуальность стандартов.

## Установка

1. Откройте страницу релиза:

   <https://github.com/dwnmf/kompas-3d-guard-bin/releases/latest>

2. Скачайте wheel:

   ```text
   kompas_3d_guard-0.5.1-py3-none-win_amd64.whl
   ```

3. Установите:

   ```bash
   pip install kompas_3d_guard-0.5.1-py3-none-win_amd64.whl
   ```

После установки доступны CLI и Python SDK.

## Быстрая проверка

```bash
kompas-guard up
kompas-guard status
kompas-guard context --task "Проверить доступность IApplication.ActiveDocument"
kompas-guard down
```

Ожидаемо в `status` должны быть `OK` для context service и local runner.
Live runner требует установленный и зарегистрированный КОМПАС-3D.

## Использование из Python

```python
from kompas_guard import Client

kg = Client(autostart=True)

health = kg.health()
print(health.ok)
print(health.feedback)

ctx = kg.context("Получить имя активного документа КОМПАС-3D")
print(ctx.feedback)
```

После установки wheel `Client()` автоматически находит встроенный compiled
runtime в:

```text
site-packages/kompas_guard/_app
```

Указывать `app_dir` обычно не нужно.

## Проверка и запуск candidate.py

Файл-кандидат должен содержать функцию:

```python
def run(app):
    # app — live IApplication COM object КОМПАС-3D
    doc = app.ActiveDocument
    return doc.Name
```

Дальше:

```python
from kompas_guard import Client

kg = Client(autostart=True)

v = kg.verify_file("candidate.py")
if not v.ok:
    print(v.feedback)
    raise SystemExit(1)

r = kg.run_file("candidate.py", setup=["kompas_running"], risk="read")
print(r.ok)
print(r.result_repr)
```

## Поддерживаемые версии Python и Windows

Текущий wheel:

```text
kompas_3d_guard-0.5.1-py3-none-win_amd64.whl
```

Подходит для:

- Windows x64;
- CPython 3.10;
- CPython 3.11;
- CPython 3.12;
- CPython 3.13;
- CPython 3.14.

Не подходит для:

- Linux;
- macOS;
- 32-битной Windows;
- Windows ARM64.

Для live-запусков нужен установленный КОМПАС-3D с зарегистрированным COM API.
Проверено на КОМПАС-3D v24.

## Что внутри wheel

Wheel содержит:

- тонкую Python SDK/CLI-обвязку `kompas_guard`;
- встроенный source-free Nuitka runtime;
- обработанные runtime-данные для API/GOST grounding.

Приватные исходники внутренних сервисов не включены:

```text
serve/
runner_service/
scripts/
```

## Skill для агентов

Skill находится здесь:

```text
.agents/skills/kompas-guard-sdk/SKILL.md
```

Используйте его, когда агент пишет или проверяет Python COM-код для КОМПАС-3D,
или когда нужны grounded-факты по KOMPAS API / ГОСТ / ЕСКД / СПДС.

## Проверка целостности

Для v0.5.1 SHA256 wheel:

```text
e368b1d9b04dcba65c5bbe6fb2d91a372fe6e84ad240737ddf1738f26cd9fa61
```

Проверить в Windows:

```cmd
certutil -hashfile kompas_3d_guard-0.5.1-py3-none-win_amd64.whl SHA256
```

## Частые команды

```bash
kompas-guard up                 # запустить context service + runner
kompas-guard status             # проверить состояние
kompas-guard doctor             # диагностика окружения
kompas-guard context --task "..."
kompas-guard verify --file candidate.py
kompas-guard run --file candidate.py --setup kompas_running
kompas-guard down               # остановить сервисы
```

## Важно

- Не запускайте `kompas-guard-runtime.exe` напрямую без необходимости.
  Используйте `kompas-guard` или Python SDK.
- Optional embed/llama.cpp сервер по умолчанию выключен. Он нужен только для
  отдельных A/B-сценариев и запускается через `kompas-guard up --with-embed`.
- Для юридически значимого применения стандартов проверяйте финальные требования
  по официальным источникам.
