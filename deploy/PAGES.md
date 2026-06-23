# Деплой на GitHub Pages (альтернатива k3s)

Нулевая инфраструктура, публично, бесплатный TLS. Лаг чуть выше (~1–2 мин).

## Переключение CI на Pages

Замени содержимое `.github/workflows/ci.yml` на публикацию в `gh-pages`:

```yaml
name: ci
on:
  push:
    branches: [main]
permissions:
  contents: write
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Configure Git Credentials
        run: |
          git config user.name github-actions[bot]
          git config user.email 41898282+github-actions[bot]@users.noreply.github.com
      - uses: astral-sh/setup-uv@v5
      - run: uv pip install --system -r requirements.txt
      - run: mkdocs gh-deploy --force
```

## Включить Pages

1. Пуш → создастся ветка `gh-pages`.
2. Settings → Pages → Source → ветка `gh-pages`.
3. Сайт: `https://USERNAME.github.io/notes/`
   (тогда в mkdocs.yml `site_url` укажи этот адрес с подпутём `/notes/`).
