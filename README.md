# dev-notes-wiki

A living wiki of my technical learning — organized by topic, built with MkDocs Material, meant for quick review later

Currently includes:
- **Backend from first principles** — notes from [this playlist](https://www.youtube.com/playlist?list=PLui3EUkuMTPgZcV0QhQrOcwMPcBCcd_Q1)

## Project structure

```
dev-notes-wiki/
├── mkdocs.yml              # site config + sidebar nav
├── requirements.txt        # pinned python deps
├── docs/
│   └── backend-from-first-principles/
│       ├── index.md
│       ├── 01-intro.md
│       ├── 02-core-concept.md
│       └── ...
└── README.md
```

## Setup

### 1. Clone the repo

```bash
git clone https://github.com/essadany/dev-notes-wiki.git
cd dev-notes-wiki
```

### 2. Create a virtual environment

```bash
python3 -m venv .venv
source .venv/bin/activate
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

### 4. Scaffold the project (first-time setup only)

Only needed once, when the project doesn't have `mkdocs.yml` / `docs/` yet:

```bash
mkdocs new .
```

If you're cloning this repo (it already exists), skip this step — `mkdocs.yml` and `docs/` are already committed.

### 5. Preview locally

```bash
mkdocs serve
```

Open `http://127.0.0.1:8000` — the site live-reloads as you edit any `.md` file.

### 6. Add a new note

Create a new `.md` file under `docs/<topic>/`, then add it to the `nav:` section in `mkdocs.yml` so it shows up in the sidebar in the right order.

### 7. Deploy to GitHub Pages

```bash
mkdocs gh-deploy
```

Builds the static site and publishes it to the `gh-pages` branch, live at `https://essadany.github.io/dev-notes-wiki`.

## License

[MIT](LICENSE)
