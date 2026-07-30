---
name: kompas-guard-sdk
description: Находить, проверять и исполнять KOMPAS-3D API7 код через локальный direct Python SDK. Использовать для поиска RU/EN API members, точных сигнатур и get/set операций, enum-констант, acquisition/cast путей, GOST/ЕСКД, статической проверки Python и явно запрошенного live-запуска с проверкой геометрической семантики. Никогда не угадывать API, координаты или числовые константы из памяти.
---

# KOMPAS Guard SDK

Работать через `KompasGuard` прямо в Python-процессе. Не запускать фоновые
сервисы и не подключать внешнюю LLM. Только `run()` создаёт один локальный
изолированный worker для управления desktop КОМПАС.

Установить protected Windows x64 wheel для обычного GIL-enabled CPython
3.10–3.14. Использовать тот Python, в котором будет работать SDK:

```powershell
python -m pip install -U kompas-3d-guard==0.6.5
```

При первом `KompasGuard()` SDK скачивает закреплённый Core snapshot, сообщает
этапы через callback `status` и проверяет SHA-256. Затем использует локальный
кэш. Не запрашивать и не встраивать Hugging Face token.

## Обязательный цикл

1. Декомпозировать задачу и вызвать **один** `context(task, queries=[...])`,
   передав все RU/EN подвопросы сразу. Один этот вызов возвращает retrieval,
   relevant enum-таблицы, доказанные acquisition paths и вердикт.
2. Прочитать последнюю строку вывода:
   - `READY|` — grounding полон. Немедленно писать candidate, не искать больше.
   - `NEED|n|...` — выполнить ровно перечисленные пункты **одним батчем**
     (`inspect_many`, `path`, `constants`), затем прекратить поиск.
   Не добавлять «на всякий случай» вызовы сверх списка `NEED`.
3. Получить числовые значения только через `enum_values`, `constant` или
   `constants`; получить application-domain таблицы через `app_const`.
   Значения, уже пришедшие в `context`, повторно не запрашивать.
4. Написать новый candidate с `def run(app)` и проверить его через
   `verify(..., backend="auto")`.
5. Если пользователь явно просит исполнение, вызвать `run()` с read-only
   `check` в той же сессии. Для созданных тестовых документов выбрать
   `cleanup="created"`.
6. При сбое вернуть compact диагностику и repair prompt Guard. Не исправлять код
   внешней моделью автоматически.

```python
from kompas_guard import KompasGuard

kg = KompasGuard(status=print)
ctx = kg.context(
    "создать деталь, эскиз на XOY, окружность, выдавливание и сохранить",
    queries=[
        "получить TopPart и model container",
        "начать редактирование эскиза и получить circles",
        "создать base extrusion и сохранить документ",
    ],
)
print(ctx.render("compact"))

# Только если последняя строка была NEED|, и только по её пунктам:
details = ctx.batch.inspect_many(["Q1.R1", "Q2.R1"])
```

## Контракт candidate

Передавать исходный Python-текст с функцией:

```python
def run(app):
    # app — early-bound KompasAPI7.IApplication
    return value
```

Использовать `cast(value, "IInterface")`, предоставляемый worker. Не
импортировать `win32com`, не открывать `gen_py`, Interop DLL или raw graph.
Возвращаемое значение должно быть компактным и сериализуемым либо иметь
безопасный `repr`.

## Поколение API: API7 и legacy API5

Bundle содержит оба поколения, и они несовместимы в одном candidate:
`app` — это `KompasAPI7.IApplication`, и объект API7 нельзя cast’ить в `ks*`
интерфейс API5. К `API5` относятся не только `ks*`, но и омонимы вроде
`IPart`, `IDocument3D`, `ISurface`.

Поиск по умолчанию ограничен API7. Явно менять только по задаче:

```python
kg.context(task, queries=[...])            # api="api7" по умолчанию
kg.batch(["штамп"], api="api5")             # осознанный legacy-поиск
kg.batch(["штамп"], api="any")              # оба поколения
```

В каждой строке выдачи есть `api=API7|API5`; `context()` печатает
`API|preferred=...|mixed=...` и `API_MIX|` для чужих результатов. Не смешивать
поколения в одном `run(app)`: `verify()` выдаёт `API5_SYMBOL_IN_API7`, а
`kg.api_lint(code)` возвращает найденные legacy-типы.

## Поиск и точные symbols

Формировать batch сразу на русском и английском, когда термин неоднозначен.
Точный запрос `IType.Member` защищает symbol от semantic перестановки.
У каждого результата сохранять `stable_ref`, symbol, operation kind,
signature, summary и acquisition path.

Если у symbol есть get и set, выбирать по действию задачи и подтверждать через
`inspect`. `path|partial:...` означает, что owner ещё не получен полностью.

## Enum и constants

Не использовать магические числа:

```python
part_doc = kg.constant("DocumentTypeEnum.ksDocumentPart")
xoy = kg.constant("ksObj3dTypeEnum.o3d_planeXOY")
planes = kg.enum_values("ksObj3dTypeEnum", contains="plane")
found = kg.constants("базовая плоскость XOY", enum="ksObj3dTypeEnum", k=5)
stamp = kg.app_const("KompasStampCellEnum")
print(xoy.render("compact"))
```

Authoritative Interop побеждает curated aliases. Поле `conflict` явно показывает
расхождение; не брать curated numeric value при конфликте. Короткое имя
константы допустимо только при однозначности.

`inspect()` показывает `related_enums` для arguments/properties, а `context()`
включает относящиеся enum-таблицы.

## Interface paths и casts

```python
print(kg.path("IModelContainer", from_type="IPart7").render("compact"))
print(kg.path("IDrawingContainer", from_type="IFragmentDocument").render("compact"))
print(kg.coclass("IPart7").render("compact"))
```

Путь состоит из `member`, `inherit`, `coclass` и `cast` steps с provenance.
Следовать steps буквально. `cast` действует на тот же COM-object; `member`
получает другой object. Не заменять member-chain прямым cast.

Acquisition chain доказывается относительно типа документа, который
подразумевает задача, а не только от `IApplication`: от `IApplication`
доказуемо лишь ~5% owner-ов, потому что cast от общего `IKompasDocument` к
конкретному типу документа несостоятелен. Отрендеренный путь всегда начинается
с имени своего корня, поэтому относительный путь виден как относительный.
`context()` выбирает корни по задаче автоматически.

Шаг вида `Add(args=1)` означает, что член требует аргументов: цепочка
доказана, но значения нужно взять из signature и enum-таблиц, а не придумать.

`cast` разрешён только там, где он верен для любого объекта исходного типа
(расширение к предку либо однозначный coclass), либо над объектом, который
объявлен как корень цепочки. Сужающий cast от базового типа (`IKompasDocument`,
`IModelObjects`, `IDrawingObject`) в середине цепочки не доказательство, а догадка
о том, чем оказался объект, и граф его не выдаст.

Для эскиза ожидается доказуемая форма:

```text
ISketch.BeginEdit → IFragmentDocument
IFragmentDocument.ViewsAndLayersManager → IViewsAndLayersManager
IViewsAndLayersManager.Views → IViews
IViews.ActiveView → IView
cast IView → IDrawingContainer
IDrawingContainer.Circles → ICircles
```

Если `PATH|found=false`, остановить генерацию проблемного участка и сообщить,
какой переход не доказан. `UNKNOWN_CAST` не смешивать с `UNKNOWN_METHOD`.

## Context и solid recipe

`context()` объединяет retrieval, общие recipes, relevant enum и path evidence.
Recipe `api7-solid-workflow-v1` задаёт общий порядок:

```text
document → top part → model container → base plane → sketch
→ edit geometry → feature → rebuild/check → save
```

Не считать recipe готовым скриптом: для каждого выбранного symbol сохранить
stable ref, signature, constants и cast provenance. Размеры и геометрия детали
в recipe отсутствуют.

## Verify

Всегда вызывать `verify()` перед выдачей исполняемого кода:

```python
proof = kg.verify(code, task=task, backend="auto")
print(proof.feedback)
```

- `auto`: compiler proof, затем консервативный graph fallback.
- `compiler`: требовать компиляцию прямолинейного Python→C# COM-кода.
- `graph`: проверять известные owners, operations, property set и casts.

`ok=true` доказывает только охваченные формы. `COMPILER_UNSUPPORTED` означает,
что сложная Python-конструкция не получила compiler proof. Исправлять только
участок, указанный строкой и diagnostic code.

## Геометрическая семантика и live-пробы

`verify()` доказывает API-форму, но не положение, связность или качество тела.
Перед сложной моделью выполнить минимальную live-пробу спорной геометрии и
прочитать значения обратно. Найденные и подтверждённые API7-инварианты:

- `ICircle.Xc/Yc` — центр; `X/Y` — точка окружности. Задавать согласованно:
  `Xc=cx, Yc=cy, X=cx+r, Y=cy, Radius=r`, затем проверять getters, `Valid`,
  `Update()` и выдавливание. Не использовать `X/Y` как центр.
- `IRectangle.X/Y` — нижний левый угол, не центр. Для центра `(cx, cy)` задавать
  `X=cx-w/2, Y=cy-h/2`; подтверждать вершины через
  `IRectangle.ContourSegmentsPoints`.
- На базовой XOZ локальная ось эскиза `+Y` направлена по глобальной `-Z`.
  Глобальной высоте `z` соответствует локальная координата `y=-z`.
  Проверять контрольные точки через `ISketch.GetPoint3D`.
- Для доказанного reverse extrusion использовать reverse side parameters:
  `Direction=dtReverse` и `SetSideParameters(False, ...)`; для normal — `True`.
  Middle-plane применять только после отдельной live-пробы.

Каждый `Update()`, `EndEdit()`, `SetSideParameters()` и `Valid` проверять явно;
при `false` бросать исключение до `SaveAs`. После построения скрыть эскизы и
проверить `RebuildModel`. Успешные COM-вызовы ещё не доказывают единое тело:
если Guard доказал путь, дополнительно проверять число тел, `OperationResult`
и `IBody7.GetGabarit`. Проверка результата должна контролировать ожидаемые
глобальные точки/габариты, а не только счётчики операций и факт сохранения.

## Live run, check и cleanup

Запускать только по явному запросу пользователя:

```python
result = kg.run(
    code,
    timeout_s=45,
    check="""def check(app, value):\n    return bool(value)""",
    cleanup="created",
)
print(kg.classify(result, task=task, code=code).feedback)
```

- `run.ok`: candidate завершился без исключения.
- `run.check_ok`: read-only `check(app)` или `check(app, value)` подтвердил
  ожидаемый результат в той же COM-сессии.
- `run.successful`: `ok` и, если check задан, `check_ok` истинны.
- `cleanup="none"`: ничего не закрывать.
- `cleanup="created_unsaved"`: закрыть только созданные несохранённые документы.
- `cleanup="created"`: закрыть только документы, созданные этим запуском.

Никогда не закрывать документы, существовавшие до snapshot worker.

## Классификация сбоев

Различать `com_not_registered`, `com_cache_corrupt`, `kompas_start_failed`,
`runner_environment_error`, `agent_python_error`, `com_call_error`,
`check_failed`, `success`.

Environment/bootstrap failures не получают RAG candidates. Для `com_call_error`
использовать candidates Guard и compact repair prompt: проверить owner,
сигнатуру и найденный путь, переписать только проблемный участок.

## GOST/ЕСКД

Сначала разрешить актуальность через `gost_current(identifier)`. Затем искать
текст через `gost_text_search(query, gost_id=...)` или раздел через
`gost_section(gost_id, heading)`. В ответе сохранять `gost_id`, заголовок,
страницу/section и source SHA. Не выдавать старый стандарт как текущий.

## Blind-test integrity

Если проверяется способность SDK, начать в чистой директории и строить candidate
только по публичным методам Guard. Запрещено читать старые examples/scripts,
`gen_py`, DLL, JSONL графа или пользоваться API-фактами из памяти. Любое такое
чтение делает blind run недействительным, даже если модель создана.

Подробный справочник методов, CLI и окружения читать в
[references/sdk-reference.md](references/sdk-reference.md).
