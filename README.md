# my-digest

Personal news digest — daily hot discussions and articles, with Chinese translations.

## Structure

- `index.html` — static shell, loads YAML and renders client-side via js-yaml
- `reader.html` — markdown article viewer, renders .md files via marked.js
- `digests/index.yaml` — manifest of all digest files
- `digests/YYYY-MM-DD.yaml` — daily digest data (summaries in EN + ZH)
- `digests/YYYY-MM-DD/*.md` — full translated articles in Chinese

## Features

- Tabbed UI (one tab per feed provider)
- EN/中文 language toggle (preference saved in localStorage)
- Last 5 days quick nav + history
- Full Chinese article translations viewable in-browser
