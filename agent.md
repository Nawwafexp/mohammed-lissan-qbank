# agent.md — Surgery QBank

Instructions for AI agents working in this project.

## The project

A General Surgery question bank web app in plain HTML, CSS and JavaScript. It has no framework, no backend and no build step. All progress is saved in the browser's localStorage under `gsqb_state_v1`.

| Path | What it is |
|---|---|
| `index.html` | Page shell |
| `css/app.css` | All styles. Theme tokens are on `:root` (dark) and `html[data-theme="light"]`. |
| `js/app.js` | All app logic (one script): state, storage, views, sessions, search, lightbox, import/export |
| `js/icons.js` | Lucide icon paths plus the custom scalpel glyph |
| `data/questions.json` | **All content**: `books`, `sections` → `topics`, and `items` (questions) |
| `data/images/` | Original schematic figures; `data/images/eaes/` holds figures cropped from the EAES manual (see "Source material"); `data/images/covers/` has the book covers |
| `fonts/` | Self-hosted Inter |
| `tools/validate_questions.py` | Checks `data/questions.json`; run it after every content change |
| `docs/mcq-authoring-prompt.md` | Stand-alone copy of the question-writing prompt below |
| `textbooks/` | The three source textbooks as PDFs. Private reference only (see below). |

**Run locally:** `python3 -m http.server 8765` in the project folder, then open http://localhost:8765. Opening `index.html` directly as a file won't work, because the app loads its data with `fetch`.

**Check content:** `python3 tools/validate_questions.py` must end with `OK`.

## Ground rules

- **Never reuse, renumber or delete question ids** (`gs-NNNN`). User progress is saved against them.
- Keep the storage schema compatible. If it must change, bump `SCHEMA` in `js/app.js` and add a step to `MIGRATIONS`.
- Treat imported backups as untrusted. Notes are sanitised in `cleanState()`; keep it that way.
- Don't add frameworks, build tools or external CDNs. Fonts and icons stay self-hosted.
- After any UI change, check the app at 1280px, 768px and 375px in both themes.

## How the code works

`js/app.js` is a single script wrapped in one function, with no modules. The flow:

1. `boot()` → `load()` reads localStorage → `migrate()` → `cleanState()` → `loadData()` fetches `data/questions.json` → `render()`.
2. `render()` draws the nav (`renderSide()`, which fills both the desktop rail `#side` and the phone tab bar `#tabbar`), then the current view into `#view`.
3. Every view is a string-building function that sets `innerHTML`. The views are `renderHome`, `renderStudy`, `renderSearch`, `renderWeak`, `renderExpl`, `renderImages`, `renderNotes`, `renderTags` and `renderTopicNotes`.
4. **Clicks** are handled by one delegated listener. Any element with `data-act="name"` runs `ACTIONS[name]`. Add new buttons the same way, not with inline handlers.
5. **Saving:** call `save()` after changing `S`. It writes after a 300 ms pause, and `flush()` runs on `pagehide`. Never write to localStorage directly.
6. **Study decks:** `S.ui.ctx` says what's being studied. Its kinds are `all`, `section`, `topic`, `book`, `tag`, `search`, `session` and `list`. `getDeck()` builds the ordered question ids, cached until `invalidate()`. Open a deck with `openCtx(ctx, itemId, { push })`. `push: true` adds a Back step.
7. **Dialogs and toasts:** use `dialog()` + `openLayer()`/`closeLayer()`, `confirmDlg()` and `toast()`. Never use `alert`/`confirm`. Esc closes one layer at a time (`escLayer()`).
8. **Rendering user text:** always pass it through `esc()`. Notes go through `sanitize()`. Question HTML in `questions.json` is trusted content.

### Saved state (`S`, key `gsqb_state_v1`, `SCHEMA = 1`)

```
answers {id: choiceIndex}   starred {id: true}   counts {id: n}   notes {id: html}
topicNotes {topicId: html}  tags {id: [..]}      days {"YYYY-MM-DD": {new, repeat}}
sessions {sid: {id, name, created, filters, ids, pos}}   goal 40
ui {view, ctx, filters:{status}, position, shuffle, seed, choiceShuffle, choiceSeed, theme, sound, haptics, ...}
```

- `counts` and `days` only feed the heatmap and streak. There's no "times answered" UI any more.
- `ui.mode`, `ui.sidebar` and `ui.collapsed` are leftovers. `ui.mode` is always `'question'`; `ui.collapsed` is still used for the session builder's folding.

## Screens

- **Navigation:** a slim icon rail on the left on desktop and tablet, and a 4-item bottom tab bar on phones. The tab bar has Home, Weak, Notes and More; More opens a sheet with Explanations, Images, Tags and Settings. Study is accessed from Home and decks rather than a navigation item. A red dot on "Weak" means there are low-accuracy topics.
- **Home:**
  1. A hero card with a greeting, Continue studying and Custom session buttons, today's ring with the daily goal input, Streak, Accuracy and Progress, and the year heatmap. **The heatmap stays at the top.**
  2. Search.
  3. Continue (paused sessions).
  4. Textbooks: covers with progress.
  5. Systems: a card per section.
- **Study:** one centred column, about 920px wide.
  1. The title, with a Browse topics button for section and topic decks. It opens a searchable dialog with question counts, answered progress and the current selection.
  2. **One control bar above the question:** question-number chips, a progress line, the count, the shuffle / reshuffle / answer-shuffle buttons, and the All / New / Correct / Incorrect / Starred filter.
  3. The question card.
  4. The Notes card.

  Previous/Next are the round arrows beside the combined question and notes area, vertically centred on that area's full height and scrolling with it. When there's no room beside it, as on phones, they appear once below the notes. **There's no other Previous/Next.**
- **Question card after answering:**
  - The right option turns green and a wrong pick turns red.
  - **Every option shows its rationale** underneath. The app prefixes "Correct." or "Incorrect.".
  - Below the options sits the **Educational objective** box, then the source line with a tiny book cover, then a Topic notes link.
  - There's no "You chose…" line and no explanation bullets.
- **Card headers:** the question card and the Notes card share the same header style: a small uppercase label with an icon, 55px tall, with tools pinned top-right at equal 13px insets.
- **Settings:**
  - Grouped cards: theme preview tiles (System / Light / Dark) and Feedback switches.
  - A "Your data" section: stat tiles, Export and Import rows, a separate red Reset area.
  - Keyboard shortcuts.
- **Keyboard:** ← → navigate, S star, R retry, C copy, / search, Esc closes one layer. Letter keys never pick answers.
- **Phone gestures:** swipe left/right across the question body for next/previous. Question navigation stops at deck boundaries. The image viewer swipes through its images when unzoomed, wrapping like its arrow buttons; zoomed images pan instead. Vertical scrolling, taps, notes editing and pinch zoom retain their native behaviour. Swipes start away from the screen edges and cancel on multi-touch.

## Design system and decisions

These were set by the user over many iterations. Keep them unless asked otherwise.

- **Font:** Inter everywhere, self-hosted in `fonts/`. The Fraunces serif and the SF/Apple system font were both tried and rejected.
- **Look:** minimal and flat, with a single violet accent (`--accent`). The theme follows the device setting by default (Settings → System). Dark is the base stylesheet; light overrides it under `html[data-theme="light"]`.
  - Dark background `#0a0a12`; light mode is a soft lavender `#f6f4ff`.
  - **Gradients and glows were tried and rejected:** keep them minimal.
  - Green means correct and red means wrong; avoid them elsewhere.
- **Colours:** sections are pink, blue and orange (`color` in `questions.json`); books have their own `color`, used to tint their shelf.
- **Rhythm:** 24px between page blocks on desktop and tablet, 16px on phones. Cards have 22–24px radius, inputs 14px, and buttons are pill-shaped.
- **Rejected looks:** a full "Apple design language" pass and a "bold & vibrant" gradient pass were both built and then reverted or toned down. Don't reintroduce them.
- **Icons:** Lucide paths in `js/icons.js`. Add new icons there, with a 2px stroke and round caps.
- **Respect `prefers-reduced-motion`.** The entrance animation (`.enter`) runs only on view changes, not on in-place re-renders.

### How `css/app.css` is organised (important)

The stylesheet grew in **layers appended at the end**, and later blocks override earlier ones. In order:

1. The original base.
2. Layout revision.
3. Design refresh.
4. Overhaul (rail and hero).
5. Bold & vibrant (holds the **current colour tokens** for both themes).
6. Minimal gradients (flattens the previous layer: `--grad` and `--ring` point at plain colours).
7. Pixel audit fixes.
8. Single-column study.
9. Settings, plus the final per-option rationale and objective styles.

When changing a style, find the **last** rule that sets it. Search the file for the selector; don't just edit the first match. Some early rules are dead, such as the old sidebar, `.tabs2`, `.tile` and `.study-side.wide`. A cleanup that merges the layers is welcome, but check the app at 1280, 768 and 375 in both themes afterwards.

## Features removed on purpose (don't re-add)

- Approach / summary cards and the Approaches/Questions toggle.
- Zen mode.
- Clue keyword highlighting and the highlighter button. `<span class="clue">` may still appear in stems but is unstyled.
- "Times answered" filters and chips.
- The edit-answer pencil.
- The exam date countdown.
- The nested topic list in a sidebar and the horizontal topic pills. Use the Browse topics dialog instead.
- The "You chose X · correct answer Y" result line.
- Explanation bullets (`explanation` field), replaced by `rationales` + `objective`.
- Duplicate Previous/Next buttons in the control bar and under the notes.

## Testing in the browser

- Serve with `python3 -m http.server 8765` and check at 1280×900, 768×1000 and 375×812, in **both** themes.
- **Don't overwrite the user's saved progress while testing.** In the page, run `Storage.prototype.setItem = function(){}` before clicking around, then reload when done. If you must change stored state, back it up first and restore it exactly.
- Quick checks: `document.documentElement.scrollWidth - document.documentElement.clientWidth` should be `0` (no sideways scroll), and the console should show no errors.

## Content status

- 9 sample questions (`gs-0001` … `gs-0045`, not consecutive), one per topic, all citing **Bailey & Love 28th**.
- 573 questions (`gs-0046` … `gs-0618`) from **Surgical Principles of Minimally Invasive Procedures (EAES 2017)**, all 40 chapters, about 98 with book figures. `source.chapter` is the printed chapter title.
- 8 sections, 49 topics: Hepatobiliary, Hernia & Abdominal Wall and Endocrine gained MIS topics; new sections are MIS Principles, Oesophagus & Stomach, Bariatric Surgery, Colon Rectum & Bowel, Innovative Techniques.
- 3 books: European Coloproctology (2nd), Surgical Principles MIS (EAES 2017) and Bailey & Love (28th). European Coloproctology has **no questions yet** and show "No questions yet" on Home.
- Coloproctology questions can reuse the "Colon, Rectum & Bowel" section; add topics as needed (see "Adding a book, section or topic" below).
- Validator warnings on the current samples are known: C is the answer 5 of 9 times, and "Primary Hyperparathyroidism" is over 24 characters.
- The three book covers in `data/images/covers/` are the publishers' cover images, used for a personal study app. The SVG figures in `data/images/` are original schematics.

- 546 questions (ids in `gs-0619`…`gs-1164`, below gs-2000) from **Bailey & Love 28th**, spread across 19 sections (new: Basic Principles, Paediatric, Perioperative, Trauma, Orthopaedics, Skin/Burns/Plastics, Head & Neck, Neurosurgery, Breast, Cardiothoracic, Vascular, Genitourinary, Transplantation). Chapters read selectively (summary boxes plus key pages); coverage is uneven and ortho, paediatrics, ENT/eye and basic-science chapters are thin. 19 carry figures: 17 crops in `data/images/bailey/` (copyrighted, user-requested, excluded in `.gitignore`) and 2 original SVGs in `data/images/diagrams/`.

## Source material: `textbooks/`

The textbooks for writing questions are stored as PDFs in `textbooks/`:

| File | Book (`books[].id`) | Pages | Size |
|---|---|---|---|
| `textbooks/DOC-20250706-WA0000..pdf` | **Bailey & Love's Short Practice of Surgery, 28th ed.** (`bailey-love`) | 1,696 | 353 MB |
| `textbooks/European Coloproctology.pdf` | **Coloproctology, 2nd ed.** (European Manual of Medicine; `european-coloproctology`) | 461 | 21 MB |
| `textbooks/Surgical Principles MIS, EAES 2017.pdf` | **Surgical Principles of Minimally Invasive Procedures**, EAES manual 2017 (`eaes-mis`) | 273 | 26 MB |

The Bailey & Love file has a generic name (`DOC-20250706-WA0000..pdf`). Its embedded title confirms it's Bailey & Love. Don't rename or move it unless the user asks.

**Reading the PDFs**

- **Never load a whole PDF at once,** especially Bailey & Love. Find the chapter's page range first, then extract only those pages.
- No PDF tools are installed by default. If your environment can't read PDFs itself, install `pypdf` (`python3 -m pip install --user pypdf`). Tell the user before installing anything.
- Find the chapter's pages from the PDF's bookmarks (outline), or from the table of contents in the first pages. Example:

  ```python
  from pypdf import PdfReader
  r = PdfReader("textbooks/European Coloproctology.pdf")
  for o in r.outline:                      # top-level bookmarks = chapters
      if not isinstance(o, list):
          print(r.get_destination_page_number(o) + 1, o.title)
  text = "\n".join(r.pages[i].extract_text() for i in range(119, 134))  # pages 120–134
  ```

- Write extracted text to a temporary file outside the project if you need to keep it. **Don't save extracted chapter text inside the project.**
- Cite the **chapter title as printed in the book** in `source.chapter`. Add `source.page` only for a page number you actually read.

**Rules for this folder**

- **It is private reference material. Never publish it.** If the app is deployed (GitHub Pages or any static host) or put in git, `textbooks/` must be excluded, for example with `.gitignore`. The books are copyrighted and the folder is about 420 MB.
- **Book figures:** the user explicitly asked for figures from the EAES MIS manual to be used in its questions (Oct 2026). Those crops live in `data/images/eaes/` (named `chNN-pPAGE_N.jpg`). They are copyrighted, so the same rule applies as for the PDFs: **exclude `data/images/eaes/` from any public deploy or git push.** For the other two books, still draw original schematics or use no image. Pick book figures whose labels don't give away the answer.
- Write questions in your own words; never copy sentences from the books (see the copyright rules below).

---

## Writing MCQs from the textbooks

When writing questions, you write single-best-answer multiple-choice questions (MCQs) for **Surgery QBank**. It's a General Surgery question bank web app. The questions are for surgical trainees preparing for exams.

All content lives in one file, `data/questions.json`, in the app's folder. Your job is to add new, accurate, exam-quality questions to that file. Base them on a textbook chapter, and follow the format below exactly.

### When asked to write questions

The user will name a textbook, chapter(s) and a number of questions. Read that chapter from the matching PDF in `textbooks/` (see "Source material" above), extracting only its pages. If the book isn't there or the pages can't be read, ask the user. Never write from memory and attribute it to the book.

### 1. Read before writing

1. Open `data/questions.json` and read the `books`, `sections` and `items` arrays.
2. Find the book in `books`. If it isn't there, add it (see §5).
3. Map the chapter's diseases to existing `sections` → `topics`. Reuse an existing topic whenever one fits. Add a topic only for a genuinely new disease (see §5).
4. Find the highest existing question number (`gs-NNNN`). New ids continue from there.

### 2. What makes a good question

Write the kind of question a consultant would set for a surgical exam.

**Content**
- Test **one important, decision-relevant idea** per question: diagnosis, next step in management, anatomy that matters in theatre, a complication, or a key investigation. Skip trivia such as eponym dates, rare percentages or author names.
- Every fact must be supported by the chapter you were given. If the chapter doesn't support something clearly, don't write it.
- Aim for a mix: roughly 40% management and next step, 25% diagnosis and investigation, 20% anatomy and operative, 15% complications.

**Stem**
- A short clinical vignette, usually 1–3 sentences: age and sex, the presentation, and the 2–3 findings that decide the answer.
- End with a clear lead-in question that ends in "?". Examples: "What is the most appropriate next step?", "Which nerve is most likely injured?"
- A strong reader should be able to answer before seeing the options.
- Avoid negatives ("Which is NOT…", "EXCEPT"), double negatives and vague words like "usually" or "may".

**Options**
- **4 options** (5 is allowed, but stay consistent within a batch). Exactly **one** best answer.
- Keep options **homogeneous**: all diagnoses, or all managements, or all nerves. Keep them similar in length and grammar.
- Distractors must be **plausible**. They should be real choices a weaker candidate might make, ideally the right answer for a slightly different scenario.
- Never use "All of the above", "None of the above", or options that overlap.
- Don't give the answer away through length, extra detail or matching wording from the stem.
- **Spread the correct-answer position.** Across a batch, use positions 0, 1, 2 and 3 roughly equally.

**Rationales (one per option)**

After a question is answered, the app shows each option's rationale under that option. It adds the label "Correct." or "Incorrect." itself.
- **Don't** start a rationale with "Correct", "Incorrect", "Yes" or "No". The app already adds that.
- 1–2 sentences, plain text, maximum ~35 words.
- **Correct option:** say *why* it is right, linking it to the deciding clues in the stem.
- **Wrong options:** say *why* it is wrong here and, where useful, *when* it would be right.

**Educational objective**
- **One sentence** with the takeaway the learner should remember, written as a rule. Example: "Cholangitis with shock or confusion needs urgent biliary decompression, with ERCP first-line."
- Don't start with "This question tests…" or "The learner should…".

**Writing style**
- Plain, clear English. UK spelling is fine; be consistent.
- Write out abbreviations the first time unless they're universal (CT, MRI, ERCP, CBD, RUQ).
- Use generic drug names.

### 3. Copyright and honesty rules (must follow)

- **Write original questions.** Never copy sentences, tables, case studies or questions from the textbook. Paraphrase the *concepts*.
- **No figures from the book.** Don't scan or reproduce textbook images. A question only gets an image if you draw an original simple schematic (SVG) and save it in `data/images/`. Otherwise leave `images` out.
- **No invented citations.** `chapter` must be the real chapter title. Only include `page` if you actually know it from the provided text.
- If something in the chapter is ambiguous or might be outdated, leave it out rather than guess. List it in your final report.

### 4. Exact format for each question

Append each question to the `items` array:

```json
{
  "id": "gs-0046",
  "type": "question",
  "topic": "inguinal-hernia",
  "stem": "<p>A 58-year-old man has a recurrent inguinal hernia after a previous open mesh repair.</p><p>What is the preferred approach for re-repair?</p>",
  "choices": [
    "Repeat open anterior mesh repair",
    "Laparoscopic posterior repair (TEP or TAPP)",
    "Shouldice tissue repair",
    "Watchful waiting"
  ],
  "answer": 1,
  "rationales": [
    "Re-operating through the scarred anterior field raises the risk of re-recurrence and nerve or cord injury.",
    "After a failed anterior repair, a posterior laparoscopic approach works through unscarred tissue and lowers re-recurrence.",
    "Tissue repairs have higher recurrence than mesh and are not used for recurrent hernias.",
    "A symptomatic recurrent hernia should be repaired; watchful waiting suits minimally symptomatic primary hernias."
  ],
  "objective": "Repair a recurrence through the opposite plane: laparoscopic after a failed open repair, open after a failed laparoscopic one.",
  "source": {
    "bookId": "bailey-love",
    "book": "Bailey & Love's Short Practice of Surgery",
    "edition": "28th",
    "chapter": "Hernias, umbilicus and abdominal wall"
  },
  "tags": ["management", "operative"]
}
```

Field rules:

| Field | Rule |
|---|---|
| `id` | `gs-` + 4 digits, continuing from the highest existing number. **Never reuse or renumber an id**, because user progress is saved against it. |
| `type` | Always `"question"`. |
| `topic` | An existing topic `id` from `sections[].topics[]`, or one you add (§5). |
| `stem` | HTML. Only `<p>`, `<b>`, `<i>`, `<sub>`, `<sup>`. Put the lead-in question in its own `<p>`. Escape `"` as `\"` inside JSON. |
| `choices` | Array of 4 (or 5) plain-text strings. No letters ("A.") — the app adds them. |
| `answer` | **0-based** index of the correct choice. |
| `rationales` | Same length and **same order** as `choices`. Plain text, no HTML. |
| `objective` | One plain-text sentence. |
| `source.bookId` | Must match a `books[].id`. |
| `source.book` / `edition` / `chapter` | Book title exactly as in `books[].title`, its edition, and the real chapter title. Optional `page` only if known. |
| `tags` | 1–3 lowercase hyphenated tags. Prefer existing ones: `high-yield`, `management`, `diagnosis`, `anatomy`, `operative`, `complications`, `emergency`, `microbiology`, `genetics`, `imaging`, `critical-care`. Use `high-yield` only for core exam facts. |
| `images` | Optional: `[{ "src": "data/images/<file>.svg", "caption": "…" }]`. Original schematics only (§3). |

Don't add an `explanation` field; the app no longer uses it.

### 5. Adding a book, section or topic (only if needed)

**New book** → add to `books`:

```json
{ "id": "short-kebab-id", "title": "Full Book Title", "short": "Short name", "edition": "2nd",
  "cover": "data/images/covers/short-kebab-id.jpg", "color": "#2563eb" }
```

The cover image is optional; the app draws a placeholder until the file exists. Don't download a cover unless the user asks.

**New topic** → add to the right section's `topics`:

```json
{ "id": "kebab-case-disease", "title": "Disease Name (≤ 24 chars)", "notes": "<p>2–4 sentence summary in your own words.</p>" }
```

The app sorts topics A→Z automatically.

**New section** (a surgical system that doesn't exist yet) → add to `sections`:

```json
{ "id": "short-id", "title": "Colon, Rectum & Anus", "icon": "layers", "color": "#f97316", "topics": [] }
```

Use an `icon` that already exists in `js/icons.js`, for example `droplets`, `layers`, `activity`, `target` or `book-open`.

### 6. Validate before you finish

Run the validator from the app folder and fix every error it reports:

```bash
python3 tools/validate_questions.py
```

It checks JSON syntax, unique ids, topic and book references, the answer index, rationale counts, the stem's lead-in, allowed HTML, and how evenly answers are spread.

Then self-review each new question:
- Could a strong candidate answer it from the stem alone, without the options?
- Is there exactly one defensible best answer?
- Is every fact supported by the chapter?
- Is everything paraphrased, with no copied sentences?
- Do the rationales skip the words "Correct" and "Incorrect"?

### 7. Report back

When done, report:
1. How many questions you added, with their ids and topics.
2. Any new books, sections or topics you created.
3. Anything you left out because the chapter was unclear, and any facts that need a clinician to double-check.
4. The validator's final output.

Don't change anything in `data/questions.json` other than adding items and, if needed, books, sections or topics. Don't edit or delete existing questions unless the user asks.


## Content status: European Coloproctology (added Oct 2026)

- 743 questions (`gs-2001` to `gs-2743`) from chapters 2-38 of *European Coloproctology* (2nd ed.), in 6 new sections (`coloproct-basic`, `anorectal`, `pelvic-floor`, `ibd`, `crc`, `colorectal-care`) and 36 topics whose ids start with `ecp-`. Chapter 1 (history of the division) has no questions.
- 51 questions carry figures cropped from the PDF into `data/images/ecp/`. Like `data/images/eaes/`, these are copyrighted: exclude them and `textbooks/` from any public deploy or git push.
- Known limits: in most items the correct option is longer than the distractors, so consider balancing option lengths; all content should be reviewed by a clinician.
