# Публикация в PyPI через Trusted Publishing

В репозитории настроен workflow:

```text
.github/workflows/publish-pypi.yml
```

Он берёт `.whl` из GitHub Release и публикует его в PyPI без хранения токена в
секретах GitHub.

## Что нужно один раз настроить на PyPI

В PyPI откройте настройки проекта `kompas-3d-guard` и добавьте Trusted Publisher.

Если проекта ещё нет, создайте pending publisher в PyPI.

Параметры:

```text
Owner: dwnmf
Repository name: kompas-3d-guard-bin
Workflow name: publish-pypi.yml
Environment name: pypi
```

После этого публикация выполняется из GitHub Actions.

## Как опубликовать текущий релиз

Откройте GitHub Actions в репозитории `kompas-3d-guard-bin`, выберите workflow
`Publish wheel to PyPI`, нажмите `Run workflow` и укажите тег:

```text
v0.5.1
```

Workflow скачает wheel из GitHub Release, проверит metadata через `twine check`
и отправит пакет в PyPI.

После успешной публикации установка будет:

```bash
pip install kompas-3d-guard
```
