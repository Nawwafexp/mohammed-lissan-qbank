# Surgery QBank

A General Surgery question bank in plain HTML, CSS and JavaScript. It has no backend and no accounts. All progress is saved in the browser's localStorage under the key `gsqb_state_v1`.

## Run it

The app loads `data/questions.json` with `fetch`, so it needs a static server. Opening `index.html` directly as a file won't work.

```bash
python3 -m http.server 8765
```

Then open http://localhost:8765. It also works as-is on GitHub Pages or any static host.

On phones, swipe left/right across a question to go next/previous. Open an image to swipe through the gallery; pinch to zoom, then drag to pan. The arrow buttons remain available.

Run the gesture regression checks with `node --test tools/test_swipes.js`.

## Single-file build

`python3 tools/build_single.py` writes `dist/index.html`, one self-contained file with the CSS, JS, font, questions and all referenced images inlined. It opens by double-click with no server. The files above stay the source; rebuild after any change.

## Files

- `index.html`: the page shell
- `css/app.css`: all styles. Theme tokens are on `:root` (dark) and `html[data-theme="light"]`.
- `js/icons.js`: Lucide icon paths plus the custom scalpel glyph
- `js/app.js`: the app logic, including state, storage, views, sessions, search and the lightbox
- `data/questions.json`: sections, topics and items
- `data/images/`: figures
- `fonts/`: self-hosted Inter

## Adding content

- Each question needs a **permanent `id`** in the form `gs-NNNN`. Never reuse or renumber ids, because progress is keyed by id.
- The number in the id becomes the Q label (`gs-0012` shows as Q012).
- Mark 2–3 deciding clues in a stem with `<span class="clue">…</span>`.
- `rationales` holds one short reason per choice, in the same order as `choices`. Each reason appears under its option after the question is answered.
- `objective` holds the one-sentence educational objective shown below the options.
- `images` takes plain paths, or `{ "src": "...", "caption": "..." }` objects.

## Changing the storage schema

Bump `SCHEMA` in `js/app.js` and add a step to `MIGRATIONS` that upgrades the old shape.

## Textbooks and covers

The home page lists questions by textbook. Books are defined in the `books` array in `data/questions.json`. Link a question to a book with `source.bookId` (for example `"bookId": "bailey-love"`). If `bookId` is missing, the app falls back to matching `source.book` against the book title, ignoring case, spacing and curly apostrophes.

To show the real covers, add the images here (JPG, portrait, around 2:3):

- `data/images/covers/european-coloproctology.jpg`
- `data/images/covers/eaes-mis.jpg`
- `data/images/covers/bailey-love.jpg`

Until a file exists, the app draws a plain coloured book placeholder instead.

## Imported backups

Imported backups are treated as untrusted. Notes are re-sanitised and every field is checked before use.

## Writing new questions

`docs/mcq-authoring-prompt.md` is a ready-to-use prompt for an AI agent that writes questions from a textbook chapter. After any change to the question file, run `python3 tools/validate_questions.py` to check it.
