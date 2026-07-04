<div align="center">
<img width="2804" height="266" alt="KOMPAS-3D_GUARD_CondensedBold" src="https://github.com/user-attachments/assets/9ed1e9b9-f19e-4e47-80f7-80f555895bfb" />

<strong>Бинарная pip-сборка KOMPAS-3D_GUARD для Windows x64</strong><br>
Python SDK и CLI для работы агентов с КОМПАС-3D, KOMPAS API и ГОСТ/ЕСКД/СПДС grounding.

<p>
  <a href="https://github.com/dwnmf/kompas-3d-guard-bin/releases/latest">Скачать последнюю версию</a>
  ·
  <a href=".agents/skills/kompas-guard-sdk/SKILL.md">Skill для агентов</a>
</p>
</div>

---

## Назначение

**KOMPAS-3D_GUARD** помогает Python-скриптам и coding-агентам безопаснее работать
с КОМПАС-3D через COM/API. Guard подсказывает реальные интерфейсы, методы,
свойства и enum'ы KOMPAS API, проверяет Python COM-код до запуска, выполняет код
в живом КОМПАС-3D через локальный runner, классифицирует COM/HRESULT/null-state
ошибки и даёт grounding по ГОСТ, ЕСКД и СПДС с учётом актуальности стандартов.

Репозиторий ориентирован на пользователей бинарной сборки. Здесь доступны wheel
для установки через `pip`, краткая документация и skill для pi/agent.

---

## Установка

Установите пакет из PyPI обычной командой `pip`:

```bash
pip install kompas-3d-guard
```

Для установки конкретного wheel-файла из GitHub Releases можно использовать:

```bash
pip install kompas_3d_guard-0.5.1-py3-none-win_amd64.whl
```

После установки становятся доступны консольная команда `kompas-guard` и Python
SDK `kompas_guard`.

---

## Быстрый старт через CLI

Для интерактивной работы через CLI удобно один раз поднять локальные сервисы:

```bash
kompas-guard up
```

Проверьте состояние:

```bash
kompas-guard status
```

Выполните тестовый запрос к API grounding:

```bash
kompas-guard context --task "Проверить доступность IApplication.ActiveDocument"
```

Остановите сервисы после работы:

```bash
kompas-guard down
```

В `status` ожидается состояние `OK` для context service и local runner. Optional
embed/llama.cpp сервер при обычном `kompas-guard up` не запускается. Для live
операций требуется установленный КОМПАС-3D с зарегистрированным COM API.

---

## Быстрый старт через Python

```python
from kompas_guard import Client

kg = Client(autostart=True)

health = kg.health()
print(health.ok)
print(health.feedback)

ctx = kg.context("Получить имя активного документа КОМПАС-3D")
print(ctx.feedback)
```

После установки wheel клиент автоматически находит встроенный compiled runtime в
каталоге установленного пакета:

```text
site-packages/kompas_guard/_app
```

В обычном сценарии параметр `app_dir` оставляют пустым.

---

## Проверка и запуск candidate.py

Файл-кандидат должен экспортировать функцию `run(app)`. Объект `app` — это live
`IApplication` COM object КОМПАС-3D.

```python
def run(app):
    doc = app.ActiveDocument
    return doc.Name
```

Проверка и запуск выполняются через SDK:

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

Для задач с документами удобно указывать setup-шаги, например
`ensure_3d_doc`, `ensure_assembly_doc`, `ensure_drawing_doc` или
`kompas_running`.

---

## Совместимость

| Компонент | Значение |
| --- | --- |
| PyPI package | `kompas-3d-guard` |
| Wheel | `kompas_3d_guard-0.5.1-py3-none-win_amd64.whl` |
| ОС | Windows x64 |
| Python | CPython 3.10, 3.11, 3.12, 3.13, 3.14 |
| CAD runtime | КОМПАС-3D с зарегистрированным COM API |
| Проверенная CAD-версия | КОМПАС-3D v24 |

---

## Состав wheel

Пакет устанавливает Python SDK/CLI-обвязку `kompas_guard`, встроенный
скомпилированный Nuitka runtime и обработанные runtime-данные для API/GOST
grounding. Пользовательский сценарий остаётся стандартным: установка через
`pip`, запуск через `kompas-guard` или импорт через `from kompas_guard import
Client`.

---

## Skill для агентов

Skill для pi/agent расположен по пути:

```text
.agents/skills/kompas-guard-sdk/SKILL.md
```

Его стоит подключать, когда агент пишет или проверяет Python COM-код для
КОМПАС-3D, работает с KOMPAS API или должен отвечать на вопросы с grounding по
ГОСТ, ЕСКД и СПДС.

---

## Проверка целостности

Для релиза `v0.5.1` SHA256 wheel равен:

```text
e368b1d9b04dcba65c5bbe6fb2d91a372fe6e84ad240737ddf1738f26cd9fa61
```

Проверка в Windows выполняется командой:

```cmd
certutil -hashfile kompas_3d_guard-0.5.1-py3-none-win_amd64.whl SHA256
```

---

## Полезные команды

| Команда | Назначение |
| --- | --- |
| `kompas-guard up` | Запустить context service и runner |
| `kompas-guard status` | Проверить состояние сервисов и КОМПАС-3D |
| `kompas-guard doctor` | Выполнить диагностику окружения |
| `kompas-guard context --task "..."` | Получить API/GOST grounding по задаче |
| `kompas-guard verify --file candidate.py` | Проверить Python COM-код до запуска |
| `kompas-guard run --file candidate.py --setup kompas_running` | Запустить candidate.py через live runner |
| `kompas-guard down` | Остановить сервисы |

---

## Примечания

Runtime запускается через `kompas-guard` или Python SDK. Optional embed/llama.cpp
сервер предназначен для отдельных A/B-сценариев и включается командой
`kompas-guard up --with-embed`. Для юридически значимого применения стандартов
финальные требования следует сверять по официальным источникам.
