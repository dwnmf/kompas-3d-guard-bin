<div align="center">
<img width="1310" height="176" alt="image" src="https://github.com/user-attachments/assets/a2d11b0b-0290-41bc-af8f-00011cc0c164" />



<strong>SDK для КОМПАС-3D под Windows x64</strong><br>
Поиск KOMPAS API, проверка Python COM-кода и grounding по ГОСТ/ЕСКД/СПДС.

<p>
  <a href="https://github.com/dwnmf/kompas-3d-guard-bin/releases/latest">Скачать последнюю версию</a>
  ·
  <a href=".agents/skills/kompas-guard-sdk/SKILL.md">Skill для агентов</a>
</p>
</div>

---

Windows x64 сборка SDK для поиска и проверки
KOMPAS-3D API. SDK не поднимает HTTP-сервер, localhost endpoints, фоновые
службы и не обращается к внешней LLM.

## Установка

KOMPAS-3D Guard 0.6.5 работает с обычным GIL-enabled CPython 3.10–3.14
на Windows x64. `pip` автоматически выбирает wheel для установленной версии
Python:

```powershell
python -m pip install --upgrade kompas-3d-guard
```

Wheel также можно скачать из
[GitHub Releases](https://github.com/dwnmf/kompas-3d-guard-bin/releases/latest):

```powershell
python -m pip install .\kompas_3d_guard-0.6.5-cp312-cp312-win_amd64.whl
```

Для Python 3.10, 3.11, 3.13 и 3.14 имя wheel содержит соответственно
`cp310`, `cp311`, `cp313` или `cp314`. Free-threaded builds и 32-битный Python
не поддерживаются.

Ретривал, константы, пути и ГОСТ работают без дополнительных компонентов.
Для compiler proof в `verify()` нужен **.NET Framework 4.7.2+** (на Windows 10/11
уже есть), а сам компилятор Roslyn `csc.exe` уже встроен в wheel — см.
[Compiler proof](#compiler-proof-net-framework-и-встроенный-csc). Для `run()`
нужен установленный КОМПАС-3D с ProgID `KOMPAS.Application.7`.

Имя пакета в PyPI остаётся `kompas-3d-guard`, а имя Python-модуля —
`kompas_guard`.

## Первый запуск и загрузка моделей

```python
from kompas_guard import KompasGuard

guard = KompasGuard()
```

При первом запуске SDK скачивает закреплённый публичный Core V2 из
`dwnmf/kompas-guard-sdk-core-v2`, показывает прогресс Hugging Face, проверяет
размер и SHA-256 каждого файла и сохраняет snapshot в стандартный cache Hugging
Face. Токен Hugging Face не требуется.

Пример вывода:

```text
KOMPAS Guard: downloading Core V2 from Hugging Face...
Fetching 41 files: 100%|██████████| 41/41
KOMPAS Guard: verifying Core V2 integrity...
KOMPAS Guard: Core V2 ready.
```

Следующие запуски используют локальный cache:

```text
KOMPAS Guard: loading Core V2 from cache...
KOMPAS Guard: verifying Core V2 integrity...
KOMPAS Guard: Core V2 ready.
```

Размер Core — около 251 МБ. Опциональный Top-3 quality pack занимает около
324 МБ и скачивается только при явном включении:

```python
guard = KompasGuard(reranker_path="top3")
```

Настройки:

```powershell
$env:KOMPAS_GUARD_CACHE="D:\KOMPAS_GUARD_CACHE"
$env:KOMPAS_GUARD_OFFLINE="1"       # использовать только существующий cache
$env:KOMPAS_GUARD_BUNDLE="D:\Core" # явно указать распакованный Core
$env:KOMPAS_GUARD_QUIET="1"         # скрыть статусные сообщения SDK
```

Приложение может направить статусы загрузки в собственный интерфейс:

```python
guard = KompasGuard(status=my_status_callback)
```

## Поиск KOMPAS API

```python
from kompas_guard import KompasGuard

guard = KompasGuard()
batch = guard.batch(
    ["сохранить текущий документ", "get circle radius"],
    k=5,
)
print(batch.render("compact"))
print(batch.inspect("Q1.R1"))
```

### Поколение API: API7 и legacy API5

Bundle содержит оба поколения KOMPAS API, и они несовместимы в одном скрипте:
`app` — это `KompasAPI7.IApplication`, объект API7 нельзя cast'ить в `ks*`
интерфейс API5. Поиск по умолчанию ограничен API7:

```python
guard.batch(["основная надпись"])                # api="api7" по умолчанию
guard.batch(["основная надпись"], api="api5")    # legacy ks* API
guard.context(task, api="any")                   # оба поколения
```

Каждый результат содержит `api_version` и `source_typelib`, compact-вывод
печатает `api=API7`, а `context()` показывает `API|preferred=...|mixed=...`.
Смешение поколений в одном кандидате ловится через `guard.api_lint(code)` и
warning `API5_SYMBOL_IN_API7` в `verify()`. Enum-константы поколения не имеют:
они живут в общих `Kompas6Constants`/`Kompas6Constants3D` и не фильтруются.

## Проверка кода и grounding для исправления

```python
check = guard.verify(
    "def run(app):\n    doc = app.ActiveDocument\n    doc.Svae()\n",
    task="сохранить текущий документ",
)
print(check.feedback)
```

SDK возвращает диагностику настоящего Interop-компилятора/графа и кандидатов
API, найденных нашим RU/EN retrieval. Он не вызывает LLM и не переписывает код
автоматически.

### Compiler proof: .NET Framework и встроенный csc

Backend `compiler` доказывает код не эвристикой, а реальной компиляцией
прямолинейного Python → C# COM-кода против Interop-сборок `KompasAPI7`,
`Kompas6API5` и `Kompas6Constants`.

Компилятор уже входит в wheel: это Roslyn `csc.exe` из
`Microsoft.Net.Compilers.Toolset` (тулсет `net472`), скачанный и проверенный
по SHA-256 на этапе сборки, расположенный в `kompas_guard/_app/compiler/`.
Ставить Visual Studio, .NET SDK или отдельный Roslyn не нужно, и SDK ничего
не докачивает в runtime. Если встроенного `csc.exe` нет, SDK откатывается на
системный `%WINDIR%\Microsoft.NET\Framework64\v4.0.30319\csc.exe`.

Требование одно: установленный **.NET Framework 4.7.2+**. На Windows 10/11
он есть в системе по умолчанию. В wheel он не входит. Не путать с .NET (Core)
8/10: netcore-вариант `csc.dll` требует отдельный runtime и здесь не
используется.

```python
guard.verify(code, task=task, backend="auto")      # compiler, затем graph fallback
guard.verify(code, task=task, backend="compiler")  # требовать компиляцию
guard.verify(code, task=task, backend="graph")     # без компилятора вообще
```

Если .NET Framework отсутствует, `backend="auto"` честно сообщает
`compiler|ok=unavailable` с кодом `COMPILER_UNAVAILABLE` и продолжает консервативной
graph-проверкой; `backend="compiler"` в этом случае не выдаёт ложное `ok=true`,
а возвращает отказ. Код `COMPILER_UNSUPPORTED` означает другое: компилятор есть,
но сложная Python-конструкция не переводится в прямолинейный C#.

Проверить окружение (глобальные аргументы идут до имени команды, например
`kompas-guard --bundle D:\Core doctor`):

```powershell
kompas-guard doctor
```

`doctor` запускает найденный компилятор и показывает его состояние:

```text
compiler|ok=true|probe=ready|path=...\kompas_guard\_app\compiler\csc.exe
```

Он также проверяет регистрацию ProgID `KOMPAS.Application.7` в реестре, но не
проверяет лицензию КОМПАС и возможность запуска в текущем desktop-сеансе.

## Запуск в живом КОМПАС

```python
result = guard.run(code, setup=["kompas_running"])
if not result.ok:
    print(guard.classify(result, task=task, code=code).feedback)
```

Только `run` создаёт один изолированный локальный процесс. Поиск, ГОСТ,
формирование контекста и проверка кода остаются прямыми in-process вызовами.
Для live execution нужны Windows, установленный КОМПАС-3D и зарегистрированный
COM ProgID `KOMPAS.Application.7`.

## Skill для агентов

Актуальный skill находится в
[`.agents/skills/kompas-guard-sdk`](.agents/skills/kompas-guard-sdk). В каждый
GitHub Release также входит `kompas-guard-sdk-skill.zip`, который можно напрямую
установить в каталог skills агента.

Skill требует искать API через SDK, проверять неоднозначные результаты, вызывать
verifier перед выдачей кода и никогда не угадывать имена KOMPAS API по памяти.

## Состав релиза

```text
kompas_3d_guard-<version>-cp310-cp310-win_amd64.whl
kompas_3d_guard-<version>-cp311-cp311-win_amd64.whl
kompas_3d_guard-<version>-cp312-cp312-win_amd64.whl
kompas_3d_guard-<version>-cp313-cp313-win_amd64.whl
kompas_3d_guard-<version>-cp314-cp314-win_amd64.whl
kompas-guard-sdk-skill.zip
SHA256SUMS.txt
```

Wheel содержит единый скомпилированный Nuitka package, публичные type stubs,
native launcher изолированного runner, встроенный Roslyn `csc.exe` (net472) для
compiler proof и не содержит приватных Python-исходников.
Модели и индексы публикуются отдельно на Hugging Face под CC BY-NC 4.0 и
закрепляются в SDK неизменяемыми commit revisions.

## Переход с 0.5 на 0.6

Версия 0.6 заменяет старую сервисную архитектуру:

```python
# удалённый API версии 0.5
Client(autostart=True)

# прямой in-process API версии 0.6
KompasGuard()
```

Команды и понятия `up`, `down`, `status`, `hosted_url`, `runner_url` и localhost
services больше не используются.
