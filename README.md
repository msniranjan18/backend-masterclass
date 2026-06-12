## MkDocs + Material Theme → GitHub Pages
URL: https://msniranjan18.github.io/backend-masterclass/

Completely free, blazing fast (static site), auto-deploys on every git push. This is the most popular setup for developer docs.

## Setup steps:

### 1. Install MkDocs + Material theme
```
pip install mkdocs-material
```

### 2. In your repo root, create mkdocs.yml
```
site_name: Backend Masterclass
docs_dir: docs

theme:
  name: material
  features:
    - navigation.tabs
    - navigation.sections
    - toc.integrate
    - search.suggest
    - content.code.copy
  palette:
    scheme: slate
    primary: indigo

nav:
  - Home: README.md
  - Languages:
    - Golang: 01-languages/01-golang.md
    - Python: 01-languages/02-python.md
  - Kubernetes:
  ...
  rest of the code
  ...
```

### 3. Preview locally (instant hot-reload)
```
mkdocs serve
# → http://127.0.0.1:8000
```

### 4. Deploy to GitHub Pages
```
mkdocs gh-deploy
# → live at https://msniranjan18.github.io/backend-masterclass
```

