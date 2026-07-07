# AGENTS.md — working on the `Coding.apkg` Anki deck

Guidance for agents editing this repository. The repo is a single Anki deck of
coding-interview flashcards: **`Coding.apkg`**. Solutions are in **C++**.

---

## 1. What `Coding.apkg` is (the file format)

`Coding.apkg` is a **zip archive**, not a binary blob. It contains exactly two members:

- `collection.anki2` — a **SQLite** database (Anki schema v11 / legacy `.anki2`).
- `media` — a JSON map of media files; it is `{}` (no media in this deck).

```bash
unzip -o Coding.apkg          # -> collection.anki2 + media
# ...edit collection.anki2...
zip -j Coding.apkg collection.anki2 media   # (or use Python zipfile, see §6)
```

### Schema facts (verified)
- One note type: **Basic** (`mid = 1481928029675`), two fields **Front**, **Back**.
- One deck (`did = 1481980511350`).
- `notes` columns: `id, guid, mid, mod, usn, tags, flds, sfld, csum, flags, data`.
  - `flds` = `Front` + `"\x1f"` + `Back` (field separator is **U+001F**).
  - `sfld` = the Front with **HTML stripped** (sort/search field).
  - `csum` = `int(sha1(stripped_front)[:8], 16)` (first-field checksum, dedupe).
  - `tags` = **space-padded**, e.g. `" binary-search array "` (empty = `""`).
- `cards` columns: `id, nid, did, ord, mod, usn, type, queue, due, ivl, factor, reps, lapses, left, odue, odid, flags, data`.
  - A new/unseen card: `type=0, queue=0, ord=0, due=max(due)+1`.
- **Preserve `notes.id` and `notes.guid`** when editing existing cards so the user's
  review scheduling/history survives re-import. Set `usn=-1` and bump `mod` on change.

---

## 2. Card types & formats

There are **two treatments**. Decide by the card's nature.

### A. Concept cards — "Implement a data structure / algorithm"
(e.g. sorts, heap, graph, trie, linked list, BST, stacks/queues.)
Keep the original **study format unchanged** and only convert code to C++:

```
Constraints → Test Cases → Algorithm → Complexity → Code → Unit Test
```
- Leave Constraints / Test Cases / Algorithm / Complexity **byte-for-byte**.
- Convert the **Code** (solution) and **Unit Test** blocks to idiomatic C++17.
- Some concept cards have **multiple** code blocks (`Code: Approach A`, `Code: Hash Map…`)
  or a **Pythonic-Code** block with explanatory prose — convert each, keep the prose.

### B. Problem cards — the "Median format" (concise recall)
Everything else. Rebuild the card into these sections, in order:

```
<Problem name>            (bold title)
Problem (crux)            2–5 bullets: inputs, what to return, key limit (e.g. required time)
Structure / properties    facts to exploit: sorted? duplicates? negatives? empty?
Key insight               the one "aha" that unlocks the approach
Invariant                 the condition that stays true + a "Links to solution:" bullet
Practice                  a link ONLY if the card cites one (e.g. LeetCode); else omit
<hr>  (Back starts here)
Solution (C++)            the highlighted, verified solution
Complexity                Time / Space
```

The gold reference is the **"Median of Two Sorted Arrays"** card.

### How to derive each Median-format section
- **Problem (crux):** compress the statement so the problem is recallable *without*
  reading the original. Include the required time bound if it rules out approaches.
- **Structure / properties:** properties of the input that the algorithm leans on
  (sorted, duplicates allowed → use `<=`, negatives, one side empty, monotonic predicate…).
- **Key insight:** the single realization that unlocks the solution (e.g. "don't merge —
  binary-search the partition on the smaller array").
- **Invariant:** the property maintained that *characterizes a correct state*, i.e. what
  you'd use to *derive* the code. Pattern-specific:
  - two pointers / sliding window → what the window always satisfies
  - binary search → the answer always lies in `[lo, hi]`; predicate monotonic
  - **DP → the precise definition of `dp[...]`** and why the transition preserves it
  - greedy → the exchange / greedy-choice property that stays optimal
  - BFS → a node's distance is final when first dequeued
  Add a bullet `<b>Links to solution:</b> …` grounding the invariant in the actual C++.
- **Practice:** use the LeetCode URL the card cites; otherwise pass none (never invent one).

---

## 3. Syntax highlighting (must match the deck)

Code is pre-rendered with **Pygments** into inline-styled HTML (no external CSS/JS — works
offline in Anki). Match the deck's palette exactly:

```python
from pygments import highlight
from pygments.lexers import CppLexer
from pygments.formatters import HtmlFormatter
from pygments.styles.default import DefaultStyle
from pygments.token import Whitespace, Comment

class DeckStyle(DefaultStyle):
    styles = dict(DefaultStyle.styles)
    styles[Whitespace] = ""              # don't grey out spaces (Pygments 2.20+)
    styles[Comment] = "italic #408080"   # deck's original comment colour

PRE = ("font-family: Menlo, Monaco, Consolas, 'Courier New', monospace; "
       "font-size: 13px; line-height: 1.4; text-align: left; padding: 10px; "
       "border-radius: 4px; overflow: auto; white-space: pre;")

def render_cpp(code):
    return highlight(code, CppLexer(),
                     HtmlFormatter(noclasses=True, style=DeckStyle, prestyles=PRE)).strip()
```
- `noclasses=True` → inline styles (self-contained).
- The **monospace `prestyles` is required**: the Basic model renders Arial, which would
  misalign code otherwise.
- Result is a `<div class="highlight"><pre …>…</pre></div>` block.

---

## 4. C++ conventions & verification

- **C++17**, standard headers only. **Do NOT use `<bits/stdc++.h>`** (missing on macOS
  clang). Define helper types (`ListNode`, `TreeNode`, `Graph`) inline so a card is
  self-contained and compiles on its own.
- **Every solution must compile and pass a test before it ships.**
  - Concept cards: the converted **Unit Test** is the harness.
  - Problem cards: write a small `main()` with `assert`s from the card's **Test Cases**
    (this harness is only for verification — it is NOT shown on the card).
  - Drop Python-only cases that don't apply in C++ (e.g. passing `None` to a value type).
- **Independent re-check** (don't just trust "it compiled"): extract the C++ back out of
  the rendered HTML and syntax-check it:
  ```python
  from bs4 import BeautifulSoup
  code = "\n\n".join(pre.get_text() for pre in
                     BeautifulSoup(back, "html.parser").select("div.highlight > pre"))
  # prepend a standard includes+`using namespace std;` preamble, then:
  #   clang++ -std=c++17 -fsyntax-only file.cpp
  ```
- For faithfulness, translate what the **original** solution does; don't silently "improve"
  scope (e.g. radix sort here is non-negative-only, matching its source & tests).

---

## 5. Reading original Python (legacy concept cards)

Old cards store code as **one `<p>` per line**; tokens are `<span>`s; indentation is
`&nbsp;` runs; blank lines are `<p><br/></p>`; stray `\u200b` (zero-width) appears.
Reconstruct the source by concatenating each `<p>`'s text with `&nbsp;`→space:

```python
def line_text(p):
    return p.get_text().replace("\xa0", " ").replace("\u200b", "").rstrip()
```
Section headers are `<p>` whose entire (bold) text equals a section name
(`Code`, `Unit Test`, `Constraints`, …). A "title" line = the whole `<p>` is one `<b>`;
a code line has only a *token* bold, so `p.get_text() != p.find('b').get_text()`.

---

## 6. Workflow: extract → edit → repackage → verify

Environment: system Python is **externally managed (PEP 668)** — use a **venv**.
```bash
python3 -m venv venv
./venv/bin/pip install pygments beautifulsoup4   # + markdown only if converting notes
# clang++ (Apple clang) is available for verification
```

Repackage + integrity + round-trip (Python `zipfile`, deterministic):
```python
import zipfile, sqlite3
with zipfile.ZipFile("Coding.apkg", "w", zipfile.ZIP_DEFLATED) as z:
    z.write("collection.anki2", "collection.anki2")
    z.write("media", "media")
# then ALWAYS re-open and verify:
con = sqlite3.connect("collection.anki2")
assert con.execute("PRAGMA integrity_check;").fetchone()[0] == "ok"
```
After writing `Coding.apkg`, **re-extract it fresh** and confirm: note/card counts,
`class="highlight"` present, no leftover Python (`nose.tools`, `%%writefile`, `<b>def</b>`,
`__future__`).

### Editing an existing card
```python
front, back = flds.split("\x1f")
# ...modify...
new_flds = new_front + "\x1f" + new_back
con.execute("UPDATE notes SET flds=?, sfld=?, csum=?, mod=?, usn=-1 WHERE id=?",
            (new_flds, strip_html(new_front), csum(new_front), now, nid))
# (sfld/csum only need recomputing when the FRONT changed)
```

### Creating a NEW card (note + card rows both required)
```python
nid = int(time.time()*1000)                 # unique
guid = base64.b64encode(os.urandom(8)).decode().rstrip("=")   # ensure unique
con.execute("INSERT INTO notes (id,guid,mid,mod,usn,tags,flds,sfld,csum,flags,data) "
            "VALUES (?,?,?,?,?,?,?,?,?,?,?)",
            (nid, guid, 1481928029675, now, -1, " tag ", front+"\x1f"+back,
             strip_html(front), csum(front), 0, ""))
due = con.execute("SELECT MAX(due) FROM cards").fetchone()[0] + 1
con.execute("INSERT INTO cards (id,nid,did,ord,mod,usn,type,queue,due,ivl,factor,"
            "reps,lapses,left,odue,odid,flags,data) VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?)",
            (nid+1, nid, 1481980511350, 0, now, -1, 0, 0, due, 0,0,0,0,0,0,0,0,""))
```
where `strip_html(s) = html.unescape(re.sub(r"<[^>]+>","",s))` and
`csum(s) = int(hashlib.sha1(strip_html(s).encode("utf-8")).hexdigest()[:8], 16)`.

### Visual preview (recommended before shipping)
Render Front `+ <hr> +` Back inside the Basic CSS and screenshot with headless Chrome:
```
.card{font-family:arial;font-size:20px;color:black;background-color:white;padding:16px;max-width:900px;}
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless --disable-gpu \
  --force-device-scale-factor=2 --window-size=940,2200 --screenshot=out.png file://…/preview.html
```

### Bulk edits (many cards)
Note ids are disjoint, so bulk work parallelizes cleanly: produce one output per note
(`out/<id>.flds`), then apply them all to a fresh copy of the DB in a single pass and
repackage once. Keep each card's C++ verified before writing its output.

---

## 7. Tags (study-by-technique)

Tags let the user study one pattern (`tag:dp`, `tag:binary-search`, …). Use a **controlled
vocabulary** so tags stay consistent; assign **2–4** grounded in the actual solution, and
**preserve existing tags** (e.g. `leetcode`). Stored space-padded (`" dp array "`).

- **Patterns:** `binary-search two-pointers sliding-window dp greedy backtracking recursion
  divide-and-conquer bfs dfs dijkstra topological-sort bit-manipulation sorting prefix-sum
  fast-slow-pointers math in-place simulation`
- **Data structures:** `array string matrix linked-list stack queue hash-map hash-set tree
  binary-tree bst trie heap priority-queue graph`

---

## 8. Conventions & gotchas

- Solutions are **C++** deck-wide (migrated from Python). New cards should be C++ too.
- Keep concept cards' non-code sections unchanged; only touch code.
- `Coding.apkg` is git-tracked, so all changes are reversible — but still verify before
  writing, and repackage from a fresh extract.
- Don't touch unrelated files (e.g. `problems_2026.csv`).
- Binary-search framing used on the BS cards: ask **first-true vs last-true**, pick the
  matching template (fixes the condition, prevents off-by-one). See the
  "Binary Search — the method (recall)" card.
