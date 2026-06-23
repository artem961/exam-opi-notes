# Конспект к экзамену

Markdown-конспект → статический сайт (MkDocs Material) → GitHub Pages.

## Быстрый старт (локально)

```bash
pip install -r requirements.txt    # или uv pip install -r requirements.txt
mkdocs serve                       # http://127.0.0.1:8000
```

Пиши билеты в `docs/` (см. `CLAUDE.md` — там соглашения и правила).

## Первый деплой на GitHub Pages

1. Подставь свой `USERNAME` в `mkdocs.yml` → `site_url`
   (`https://USERNAME.github.io/notes/`; если репо назван не `notes` — поправь и путь).
2. Создай репозиторий на GitHub и запушь:
   ```bash
   git init && git add . && git commit -m "init"
   git branch -M main
   git remote add origin git@github.com:USERNAME/notes.git
   git push -u origin main
   ```
3. Пуш запустит CI (`.github/workflows/ci.yml`): он соберёт сайт и запушит
   статику в ветку `gh-pages`.
4. В репозитории: **Settings → Pages → Source → ветка `gh-pages`**.
5. Через ~минуту сайт живёт на `https://USERNAME.github.io/notes/`.

Дальше каждый `git push` в `main` пересобирает и публикует автоматически.

## Структура

```
docs/          конспект (Markdown)
mkdocs.yml     конфиг сайта
deploy/PAGES.md  заметки по деплою
scripts/       хелперы
CLAUDE.md      правила для агента Claude Code
ECOSYSTEM.md   контекст по версии MkDocs (почему запинено)
```
