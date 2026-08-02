# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

TermExtrax is a local web app for translators/localizers: upload or paste
source + target text, segment into sentences, align them (Gale–Church or
sequential 1:1), fix the alignment interactively, extract terminology from
the source, map target equivalents, and export a translation memory
(TMX/CSV) and termbase (TBX/CSV). It ships two ways: run backend + frontend
from source during development, or build a single self-contained executable
for end users (no Python/Node required at runtime).

Stack: FastAPI + spaCy backend, React (TypeScript) + Vite + Tailwind v4 +
Zustand frontend. No test suite and no linter config exist in this repo
currently — verify changes by running the app (see below).

## Commands

### Backend (from `backend/`)

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python -m spacy download en_core_web_sm   # optional, improves quality for that language
uvicorn app.main:app --reload --port 8000
```

There is no backend test suite. To sanity-check changes, hit the endpoints
directly (`curl http://127.0.0.1:8000/api/health`) or exercise them through
the frontend dev server below.

### Frontend (from `frontend/`)

```bash
npm install
npm run dev        # Vite dev server on :5173, proxies /api -> :8000
npm run build       # tsc -b (typecheck) + vite build -> frontend/dist
npm run preview     # serve the production build locally
```

`npm run build` is the closest thing to a typecheck/lint gate — it runs
`tsc -b` with `strict`, `noUnusedLocals`, and `noUnusedParameters` on before
bundling, so a broken build usually means a type error worth fixing.

### Standalone executable

```bash
./build_exe.sh       # Linux/macOS -> backend/dist/TermExtrax
build_exe.bat         # Windows (run on Windows; PyInstaller does not cross-compile)
```

Both scripts: build the frontend, `pip install -r backend/requirements.txt
pyinstaller`, copy `frontend/dist` to `backend/static`, then run
`pyinstaller TermExtrax.spec` from `backend/`. The CI workflow
(`.github/workflows/build-windows-exe.yml`) does the same on `windows-latest`
for releases/tags/manual dispatch, bundling `en_core_web_sm`, and smoke-tests
the exe by polling `/api/health`.

## Architecture

### Backend (`backend/app/`)

- `main.py` — FastAPI app and all routes: `/api/upload`, `/api/languages`,
  `/api/segment`, `/api/extract-terms`, `/api/export/corpus/{csv,tmx}`,
  `/api/export/terms/{csv,tbx}`. Also contains `_static_dir()`, which decides
  whether to serve a built frontend as a mounted SPA (checked in order:
  `TERMEXTRAX_STATIC` env var, PyInstaller's `sys._MEIPASS`, `backend/static`,
  `frontend/dist`) — this is what lets the same FastAPI app run standalone
  (packaged exe) or API-only (dev mode, frontend served separately by Vite).
- `nlp.py` — owns the spaCy pipeline cache (`get_pipeline`, `lru_cache`d per
  language) and `SUPPORTED_LANGUAGES`. Each language has an optional
  pretrained model; if it's not installed, `get_pipeline` silently falls back
  to `spacy.blank(lang)` + a rule-based sentencizer, so every language in the
  dropdown works with no setup. `extract_terms` layers three signals — noun
  chunks (only when the loaded pipeline has a parser), stopword-filtered
  unigrams, and bigrams that don't cross a sentence boundary — then merges
  counts case-insensitively and drops low-frequency unigrams that are subsumed
  by a more frequent bigram.
- `alignment.py` — `align_gale_church` is a DP over sentence-length costs
  (bead types 1-1/1-0/0-1/2-1/1-2/2-2, from the 1993 paper) with backtracking;
  `align_sequential` is the naive 1:1 fallback used both as a fallback path
  and as a selectable method.
- `office.py` — extracts plain text from `.docx`/`.xlsx`/`.pptx` using only
  the stdlib (`zipfile` + `xml.etree`), since these formats are just zipped
  XML. Preserves structure as blank-line-separated paragraphs (Word
  paragraphs, Excel rows, PowerPoint slide text blocks) so the segmenter's
  hard paragraph-boundary logic applies per-row/per-slide. Legacy binary
  formats (`.doc`/`.xls`/`.ppt`) are explicitly rejected with a message
  telling the user to re-save in the OOXML format.
- `exports.py` — builds CSV/TMX/TBX output strings from segment/term dicts.
- `schemas.py` — all Pydantic request/response models; this is the contract
  between frontend `api.ts` and backend routes — keep both in sync when
  changing a shape.
- `launcher.py` — the PyInstaller entry point. Picks a free port, starts
  uvicorn serving `app.main:app` (which self-mounts the static frontend if
  present), opens the default browser unless `TERMEXTRAX_NO_BROWSER=1`.

### Frontend (`frontend/src/`)

- `store.ts` — the only state container (Zustand), no other global state.
  The alignment table is modeled as **two independent columns** (source cells,
  target cells) rather than as rows with fixed source/target pairs. Every
  row operation (`mergeDown`, `splitCell`, `shiftUp`, `shiftDown`) mutates one
  column only, then `rebuild()` re-zips the two columns back into rows,
  padding the shorter column with empty strings and preserving row `id`s
  where content is unchanged. This column-wise model is what lets a single
  "shift" operation fix a whole misalignment cascade instead of needing one
  shift per row. `editCell`/`shiftDown`/etc. mark a row's `status` as
  `"manual_edit"` unless nothing actually changed.
- `api.ts` — thin fetch wrappers matching the backend's Pydantic schemas
  exactly (snake_case request bodies, camelCase `AlignedSegment`/
  `ExtractedTerm` fields per `types.ts`). Export calls trigger a real browser
  download by reading the `Content-Disposition` filename off the response.
- `components/` — `IngestPanel` (upload/paste + language pickers + segment
  trigger), `AlignmentTable` (renders/edits `store.segments`), `TermPanel`
  (renders/edits `store.terms`), `ExportPanel` (calls the export endpoints).
- Dev server proxies `/api/*` to `http://127.0.0.1:8000` (`vite.config.ts`) —
  the backend must be running separately for `npm run dev` to work.

### Data flow

1. Frontend uploads/pastes text → optionally `/api/upload` (Office/encoding
   extraction) → `/api/segment` (sentence split both sides + align) →
   `/api/extract-terms` (source-only) — populates `store.segments`/`terms`.
2. User edits alignment/terms entirely client-side in the store; nothing is
   persisted server-side between requests (the API is stateless/per-call).
3. Export sends the current `segments`/`terms` back to the backend, which
   serializes to CSV/TMX/TBX and returns it as a file attachment.

## Conventions worth preserving

- Backend stays stdlib-plus-FastAPI-plus-spaCy — `office.py` deliberately
  avoids adding a dependency like `python-docx`/`openpyxl` since the PyInstaller
  bundle already needs to carry spaCy's ML dependencies.
- Every language in `nlp.SUPPORTED_LANGUAGES` must keep working even without
  its pretrained model installed (blank pipeline + sentencizer fallback) —
  don't add a language-dependent feature that hard-requires a full model.
- The frontend's `AlignedSegment`/`ExtractedTerm` field names (camelCase) and
  the backend's Pydantic schemas (snake_case request fields) are two
  different naming conventions by design — match the existing one in
  whichever file you're editing rather than "fixing" the mismatch.
