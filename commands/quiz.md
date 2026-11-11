---
description: Studiequiz for et vilkårligt fag. Løbende eller én quiz nu, valgfrit antal spørgsmål og interval. Claude laver en ny statisk HTML-quiz pr. batch, svar tjekkes i browseren, out of scope huskes, næste batch N min efter færdig. Ingen installation, virker på Windows og Mac.
argument-hint: <fag> [læringsmål og emner…]  |  stop <fag>  |  list
---

# /quiz: recurring study quiz (static HTML, self-contained)

Arguments: `$ARGUMENTS`

How it works: each subject has a folder with a fixed `quiz.html`, which loads `questions.js`. Per batch, Claude writes only a new `questions.js` and opens the page. When the user presses "Færdig", the browser downloads `quiz-<slug>-b<N>.json` to Downloads. `wait.sh` picks it up, and Claude updates the history and makes the next batch. Requirements: only Claude Code (bash: Git Bash on Windows) and a browser. Chrome/Edge/Brave gives a popup window; other browsers give a normal tab.

Folders: `~/tools/quiz/_template/` (the template) and `~/tools/quiz/<slug>/` (per subject). Use `$HOME` and never hardcode a username.

## Bootstrap (ALWAYS first)
Template version: **e105ebc3**
1. If `~/tools/quiz/_template/VERSION` does not contain `e105ebc3`: write the 3 files from the **Appendix** byte for byte (Write tool, exact content between the `~~~~` fences) to `~/tools/quiz/_template/`, then write `e105ebc3` to `VERSION`.
2. Existing subject folders with an older `quiz.html`/`open.sh`/`wait.sh`: copy the new ones over them. NEVER touch `config.json`, `goals.md`, `state.json`, `questions*.js` or `results/`.

## Arguments
- `list`: for each folder under `~/tools/quiz/` (except `_template`): title, number of batches, hit rate from `state.json.history`, out-of-scope count. Stop.
- `stop <subject>`: TaskStop this session's background tasks for the subject (TaskList). Stop.
- Otherwise: first word = subject name → `slug` (lowercase, hyphens). The rest = learning goals, topics, wishes, and possibly settings ("én quiz", "15 spørgsmål", "hver 10. min").

## New subject (folder does not exist)
1. If no learning goals came with the command: ask in plain text, END the turn and wait. Don't guess. Ask for:
   - **Læringsmål** (paste them directly)
   - **Fokusemner** (optional)
   - **Ønsker** (optional: language (Danish with English technical terms), share of "for dummies" questions (~1/3))
   - **Undervisningsmateriale** (optional): paths to files or folders (PowerPoint `.pptx`, `.pdf`, `.docx`, `.md`/`.txt`), links, or just **names of books** (title + author, e.g. "Enterprise Integration Patterns – Hohpe & Woolf")
2. Create the folder and copy `quiz.html`, `open.sh` and `wait.sh` from `_template`.
3. `config.json`: `{"title": "...", "slug": "...", "mode": "loop", "questions_per_batch": 10, "delay_minutes": 5, "language": "Dansk med engelske fagtermer", "focus_topics": [], "wishes": [], "materials": [], "downloads_dir": "~/Downloads"}`
4. `goals.md`: the learning goals verbatim. `state.json`: `{"history": [], "out_of_scope": [], "batches": 0}`.
5. If material was given: see **Materials** below.
6. On first-time setup: tell the user in one line that the result file ends up in Downloads, and that the browser must not be set to "ask where to save" (otherwise a save dialog appears per quiz).
7. Continue to **Session settings**.

## Materials (optional)
`config.materials` = a list of `{"type": "file"|"folder"|"url"|"book", "ref": "<path, URL or 'Title – Author'>", "label": "short name, e.g. 'Slides uge 3'", "digested": false}`. Store absolute paths, and do NOT copy the files.
Material is read ONCE and condensed into `materials.md`, so batches stay cheap. For each entry with `digested: false`:
- **pdf:** the Read tool (use `pages` for >10 pages, read in chunks of up to 20 pages).
- **pptx/docx:** these are zip files. Extract the text per slide in numeric order and strip the XML tags:
  `for s in $(unzip -Z1 "<file>" | grep -E '^ppt/slides/slide[0-9]+\.xml$' | sort -V); do echo "--- $s"; unzip -p "<file>" "$s" | sed -e 's/<[^>]*>/ /g' | tr -s ' '; done`
  docx: `unzip -p "<file>" word/document.xml | sed -e 's/<[^>]*>/ /g' | tr -s ' '`. Use Read if the material is an image or a scanned PDF.
- **folder:** process the supported files in it (not recursively in big trees: ask if there are >30 files).
- **url:** fetch it if a web tool is available. Otherwise ask the user to download it as a file.
- **book (name only):** don't fetch anything. Use your own knowledge of the book (central concepts, chapter themes, terminology). If you don't know the book well, say so to the user, and use only the title as a pointer to the topic. Never make up page numbers or quotes.
**Material folder on the desktop (optional):** when material is given, ask (AskUserQuestion, header "Mappe") "Lav en mappe på skrivebordet med alt materialet?" Ja/Nej. On Ja: create `<Desktop>/<Fagnavn> – materiale/` (Windows: `powershell -NoProfile -Command "[Environment]::GetFolderPath('Desktop')"`, since it may be in OneDrive; macOS: `~/Desktop`). Copy the given files and folders into it. For each book chapter, create an EMPTY placeholder file `Bøger/<Bogtitel> – kap. <N> <kapiteltitel>.txt` (for a book without a chapter: `Bøger/<Bogtitel>.txt`). Links go in `links.md`. Then point `materials` at that folder.
**itslearning:** if the user wants material from itslearning, use the `itslearning` skill. It fetches files, slides and the plans' homework (`lektier.md`) and links them to the subject.

Append to `materials.md` a section per source: `## <label>` with the key concepts, definitions, terms, examples and typical exam topics, compressed (about 1–3 KB per source). Then set `digested: true`. If a file can't be read, tell the user and skip it.

## Existing subject
New goals, wishes or material in the arguments (e.g. "tilføj materiale ~/Downloads/uge4.pptx" or "fjern bogen X") → update `goals.md`/`config.json`/`materials.md` (digest only the new material). If `results/b<N>.json` is missing for the current batch N (from `questions.js`), the batch is unfinished: open it and start the waiter (event loop step 1), and skip Session settings. Otherwise continue to **Session settings**.

## Session settings (asked EVERY time a new quiz is started)
Ask with ONE AskUserQuestion call containing the questions below. Skip any question the arguments already answer clearly. Mark the value from `config.json` as "(Recommended)" / the first option, so a single click repeats last time's choice:
- **Type** (header "Type"): "Løbende quizzer" (a new quiz N min after each finished one) / "Én quiz nu" (one quiz, then stop).
- **Antal** (header "Antal"): number of questions: 5 / 10 / 15 / 20.
- **Interval** (header "Interval"), ONLY if Type = loop, or ask together and ignore it on "Én quiz nu": pause after a finished quiz: 2 / 5 / 10 / 15 min.
Save the choices in `config.json` (`mode`: `"loop"` | `"once"`, `questions_per_batch`, `delay_minutes`), so they become the default next time. Then generate the batch and open it.

## Generate a batch → `questions.js` (or `questions.next.js` while waiting)
Read `config.json`, `goals.md`, `state.json` and (if it exists) `materials.md` first. Do NOT reread the original material files per batch. The file is JavaScript:
```js
window.QUIZ = {"slug": "...", "title": "...", "batch_id": 5, "mode": "loop|once", "delay_minutes": 5,
  "note": "optional: short status, e.g. 'Sidst 7/9. Fokus nu: topic exchange, Saga.'",
  "out_of_scope": [/* copy of state.out_of_scope, so the user can undo */],
  "questions": [{"id": "b5q1", "topic": "...", "goal": "short learning goal", "type": "single|multi",
                 "q": "text, `code` allowed, \n• bullets allowed", "options": ["..."], "correct": [2],
                 "explanation": "teaches something extra, not just the answer; \n• allowed"}]};
```
Rules:
- `questions_per_batch` questions, only `single`/`multi`. `correct` = exact indices into `options`. Vary the position of the correct answer (the page also shuffles).
- **"For dummies":** ~1/3 (3–4 of 10) very simple basic questions, one concept each ("Hvad er en X?"), placed first. The rest are concrete scenario questions.
- The language from config. Technical terms in English.
- With homework (`[LEKTIE: …]` sections in `materials.md`, or `lektier.md`): every homework topic must be covered by questions over time, linked to the learning goals. Give priority to the most recent plans' homework.
- With material: base a large share of the questions on `materials.md` (the course's own terminology and examples), and end the `explanation` with `Kilde: <label>` (e.g. "Kilde: Slides uge 3" or "Kilde: EIP-bogen"). Learning goals still determine WHAT is covered; the material determines HOW it is phrased.
- Cover the learning goals broadly. Weight `focus_topics`/`wishes`. Topics answered wrong (`history.correct == false`) come back first.
- Questions MAY come back, but never word for word: rephrase, new scenario, or another aspect.
- `out_of_scope`: `scope: "topic"` → never that topic again. `scope: "question"` → not that question or close variants of it.
- Validate: `node -e` is not guaranteed, so instead check by reading the file back and making sure the JSON part after `window.QUIZ = ` is valid (e.g. `python3 -c` if available; otherwise read it carefully).

## Event loop (background, no cron)
1. Open and wait: run `bash <dir>/open.sh`, then in the background (Bash `run_in_background`): `bash <dir>/wait.sh <N>`.
2. When the waiter exits with `RESULT <path>`: read the result file `{answers:[{qid,topic,chosen,correct}], out_of_scope:[{qid,topic,question,scope}], unoos:[indices]}`:
   - Remove `state.out_of_scope[i]` for every i in `unoos` (the indices refer to the list that was in that batch's `questions.js`). Then append the new `out_of_scope` entries.
   - Append `answers` to `state.history` with `batch_id`. Set `batches = N`.
   - **mode = "once":** STOP here. Give a short status (score, weak topics) and say that `/quiz <subject>` starts a new one, which will take this result into account.
   - **mode = "loop":** write the next batch to `questions.next.js` NOW, then start ONE background command: `cd <dir> && sleep <delay_minutes*60> && mv questions.next.js questions.js && bash open.sh && bash wait.sh <N+1>`. Give the user a short status: score, weak topics, what the next batch focuses on.
3. The loop runs as long as the Claude session is open. `/quiz <subject>` in a new session resumes.

---

# Appendix: template files (write them byte for byte)

## `quiz.html`
~~~~html
<!doctype html>
<html lang="da">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Quiz</title>
<style>
  :root {
    --bg: #f6f5f1; --card: #ffffff; --ink: #1d1d1b; --muted: #6b6a65; --line: #e2e0d8;
    --accent: #2f5d8a; --ok: #2e7d4f; --ok-bg: #e6f3eb; --bad: #b3402e; --bad-bg: #fbeae6;
    --oos: #8a6d2f; --oos-bg: #f7f0e0;
  }
  @media (prefers-color-scheme: dark) {
    :root {
      --bg: #17181a; --card: #212226; --ink: #ecebe6; --muted: #9d9b94; --line: #34353a;
      --accent: #7fa8d4; --ok: #6cc391; --ok-bg: #1d3327; --bad: #e88672; --bad-bg: #3a221d;
      --oos: #d4b56f; --oos-bg: #352d1b;
    }
  }
  * { box-sizing: border-box; }
  body { margin: 0; background: var(--bg); color: var(--ink); font: 15px/1.5 system-ui, -apple-system, "Segoe UI", sans-serif; }
  header { position: sticky; top: 0; background: var(--bg); padding: 14px 18px 10px; border-bottom: 1px solid var(--line);
           display: flex; justify-content: space-between; align-items: baseline; gap: 12px; z-index: 2; }
  header h1 { font-size: 17px; margin: 0; }
  header .meta { color: var(--muted); font-size: 13px; white-space: nowrap; }
  main { padding: 14px 18px 40px; max-width: 760px; margin: 0 auto; }
  .card { background: var(--card); border: 1px solid var(--line); border-radius: 10px; padding: 16px; margin-bottom: 14px; }
  .card.done-ok { border-color: var(--ok); }
  .card.done-bad { border-color: var(--bad); }
  .card.oos { opacity: .6; border-style: dashed; }
  .card.note { border-left: 4px solid var(--accent); white-space: pre-wrap; }
  .tag { display: inline-block; font-size: 12px; color: var(--accent); border: 1px solid var(--line); border-radius: 99px; padding: 1px 9px; }
  .goal { color: var(--muted); font-size: 12px; margin-top: 4px; }
  .q { font-weight: 600; margin: 10px 0; white-space: pre-wrap; }
  .hint { color: var(--muted); font-size: 13px; margin-bottom: 6px; }
  label.opt { display: flex; gap: 10px; align-items: flex-start; padding: 8px 10px; border: 1px solid var(--line); border-radius: 8px; margin: 6px 0; cursor: pointer; }
  label.opt input { margin-top: 4px; }
  label.opt.right { background: var(--ok-bg); border-color: var(--ok); }
  label.opt.wrong { background: var(--bad-bg); border-color: var(--bad); }
  .row { display: flex; gap: 8px; flex-wrap: wrap; margin-top: 10px; }
  button { font: inherit; padding: 7px 14px; border-radius: 8px; border: 1px solid var(--line); background: var(--card); color: var(--ink); cursor: pointer; }
  button.primary { background: var(--accent); border-color: var(--accent); color: #fff; }
  @media (prefers-color-scheme: dark) { button.primary { color: #10161d; } }
  button.ghost { color: var(--oos); font-size: 13px; }
  button:disabled { opacity: .5; cursor: default; }
  .feedback { margin-top: 12px; padding: 10px 12px; border-radius: 8px; }
  .feedback.ok { background: var(--ok-bg); color: var(--ok); }
  .feedback.bad { background: var(--bad-bg); color: var(--bad); }
  .feedback .expl { color: var(--ink); margin-top: 6px; white-space: pre-wrap; }
  .oosnote { margin-top: 10px; padding: 8px 12px; border-radius: 8px; background: var(--oos-bg); color: var(--oos); font-size: 13px; }
  .summary { text-align: center; }
  .summary .score { font-size: 32px; font-weight: 700; }
  details { margin-top: 18px; color: var(--muted); font-size: 13px; }
  details li { margin: 4px 0; }
  code { font-family: ui-monospace, Consolas, Menlo, monospace; font-size: 13px; }
</style>
</head>
<body>
<header><h1 id="title">Quiz</h1><span class="meta" id="meta"></span></header>
<main id="main"><p>Indlæser… (mangler <code>questions.js</code>?)</p></main>
<script src="questions.js"></script>
<script>
const Q = window.QUIZ;
const status = {};      // qid -> {state: "ok"|"bad"|"oos", chosen, scope}
const unoos = new Set(); // indices into Q.out_of_scope the user wants restored
const KEY = Q ? `quiz:${Q.slug}:b${Q.batch_id}` : "";

const esc = s => String(s).replace(/[&<>"]/g, c => ({ "&": "&amp;", "<": "&lt;", ">": "&gt;", '"': "&quot;" }[c]));
const fmt = s => esc(s).replace(/`([^`]+)`/g, "<code>$1</code>");
const save = () => { try { localStorage.setItem(KEY, JSON.stringify({ status, unoos: [...unoos] })); } catch (e) {} };

// Stable per-question shuffle: random order, identical across reloads.
function shuffled(q) {
  let h = 2166136261;
  for (const c of q.id + Q.batch_id) h = Math.imul(h ^ c.charCodeAt(0), 16777619);
  const rnd = () => { h = Math.imul(h ^ (h >>> 15), 2246822507); h = Math.imul(h ^ (h >>> 13), 3266489909); return ((h ^= h >>> 16) >>> 0) / 4294967296; };
  const idx = q.options.map((_, i) => i);
  for (let i = idx.length - 1; i > 0; i--) { const k = Math.floor(rnd() * (i + 1)); [idx[i], idx[k]] = [idx[k], idx[i]]; }
  return idx;
}

function render() {
  document.title = Q.title;
  document.getElementById("title").textContent = Q.title;
  document.getElementById("meta").textContent = `Batch ${Q.batch_id} · ${Q.questions.length} spørgsmål`;
  const main = document.getElementById("main");
  main.innerHTML = Q.note ? `<div class="card note">${fmt(Q.note)}</div>` : "";
  Q.questions.forEach((q, i) => {
    const multi = q.type === "multi";
    const card = document.createElement("div");
    card.className = "card"; card.id = "card-" + q.id;
    card.innerHTML = `
      <span class="tag">${esc(q.topic)}</span>
      ${q.goal ? `<div class="goal">Læringsmål: ${esc(q.goal)}</div>` : ""}
      <div class="q">${i + 1}. ${fmt(q.q)}</div>
      <div class="hint">${multi ? "Vælg alle korrekte svar" : "Vælg ét svar"}</div>
      ${shuffled(q).map(j => `
        <label class="opt" id="opt-${q.id}-${j}">
          <input type="${multi ? "checkbox" : "radio"}" name="q-${q.id}" value="${j}">
          <span>${fmt(q.options[j])}</span>
        </label>`).join("")}
      <div class="row">
        <button class="primary" data-act="check">Tjek svar</button>
        <button class="ghost" data-act="oos-q">Out of scope (spørgsmål)</button>
        <button class="ghost" data-act="oos-t">Out of scope (hele emnet)</button>
      </div>
      <div class="fb"></div>`;
    card.querySelector('[data-act="check"]').onclick = () => check(q);
    card.querySelector('[data-act="oos-q"]').onclick = () => markOos(q, "question");
    card.querySelector('[data-act="oos-t"]').onclick = () => markOos(q, "topic");
    main.appendChild(card);
  });
  main.insertAdjacentHTML("beforeend", `<div class="card summary" id="summary" style="display:none"></div><details id="oos-list"></details>`);
  renderOosList();
}

function lock(q) {
  document.querySelectorAll(`#card-${q.id} input, #card-${q.id} button`).forEach(el => el.disabled = true);
}

function check(q, restored) {
  const chosen = restored || [...document.querySelectorAll(`[name="q-${q.id}"]:checked`)].map(el => +el.value);
  if (!chosen.length) return;
  document.querySelectorAll(`[name="q-${q.id}"]`).forEach(el => el.checked = chosen.includes(+el.value));
  const right = new Set(q.correct);
  const ok = chosen.length === right.size && chosen.every(c => right.has(c));
  q.options.forEach((_, j) => {
    const lab = document.getElementById(`opt-${q.id}-${j}`);
    if (right.has(j)) lab.classList.add("right"); else if (chosen.includes(j)) lab.classList.add("wrong");
  });
  lock(q);
  status[q.id] = { state: ok ? "ok" : "bad", chosen };
  const card = document.getElementById("card-" + q.id);
  card.classList.add(ok ? "done-ok" : "done-bad");
  card.querySelector(".fb").innerHTML = `<div class="feedback ${ok ? "ok" : "bad"}"><strong>${ok ? "Korrekt" : "Ikke helt"}</strong><div class="expl">${fmt(q.explanation || "")}</div></div>`;
  if (!restored) save();
  maybeSummary();
}

function markOos(q, scope, restored) {
  lock(q);
  status[q.id] = { state: "oos", scope };
  const card = document.getElementById("card-" + q.id);
  card.classList.add("oos");
  card.querySelector(".fb").innerHTML = `<div class="oosnote">Out of scope (${scope === "topic" ? "hele emnet: " + esc(q.topic) : "dette spørgsmål"}). Fremtidige batches undgår det.</div>`;
  if (!restored) save();
  maybeSummary();
}

function renderOosList() {
  const list = Q.out_of_scope || [];
  const d = document.getElementById("oos-list");
  d.innerHTML = `<summary>Tidligere out of scope (${list.length})</summary><ul>` + list.map((o, i) =>
    `<li>${o.scope === "topic" ? "<b>Emne:</b> " + esc(o.topic) : esc(o.question)}
     <button class="ghost" onclick="toggleUnoos(${i})">${unoos.has(i) ? "Fortrudt ✓" : "Fortryd"}</button></li>`).join("") + "</ul>";
}
function toggleUnoos(i) { unoos.has(i) ? unoos.delete(i) : unoos.add(i); save(); renderOosList(); }

function maybeSummary() {
  if (!Q.questions.every(q => status[q.id])) return;
  const answered = Q.questions.filter(q => status[q.id].state !== "oos");
  const right = answered.filter(q => status[q.id].state === "ok").length;
  const sum = document.getElementById("summary");
  sum.style.display = "block";
  sum.innerHTML = `
    <div class="score">${right} / ${answered.length}</div>
    <div>${answered.length && right === answered.length ? "Flot." : "Forkerte emner kommer igen i ny form."}</div>
    <div class="row" style="justify-content:center"><button class="primary" id="finish">Færdig: gem resultat og luk</button></div>
    <div class="hint" style="margin-top:8px">Resultatet gemmes i din Overførsler-mappe. ${Q.mode === "once" ? "Næste quiz tager højde for dine svar." : `Næste batch åbner ${Q.delay_minutes ?? 5} min efter.`}</div>`;
  document.getElementById("finish").onclick = finish;
  sum.scrollIntoView({ behavior: "smooth" });
}

function finish() {
  const result = {
    slug: Q.slug, batch_id: Q.batch_id, finished_at: new Date().toISOString(),
    answers: Q.questions.filter(q => status[q.id].state !== "oos")
      .map(q => ({ qid: q.id, topic: q.topic, chosen: status[q.id].chosen, correct: status[q.id].state === "ok" })),
    out_of_scope: Q.questions.filter(q => status[q.id].state === "oos")
      .map(q => ({ qid: q.id, topic: q.topic, question: q.q, scope: status[q.id].scope })),
    unoos: [...unoos]
  };
  const a = document.createElement("a");
  a.href = URL.createObjectURL(new Blob([JSON.stringify(result, null, 2)], { type: "application/json" }));
  a.download = `quiz-${Q.slug}-b${Q.batch_id}.json`;
  document.body.appendChild(a); a.click();
  document.getElementById("finish").disabled = true;
  document.getElementById("finish").textContent = "Gemt ✓";
  setTimeout(() => window.close(), 1500);
}

if (Q) {
  render();
  try {
    const saved = JSON.parse(localStorage.getItem(KEY) || "null");
    if (saved) {
      (saved.unoos || []).forEach(i => unoos.add(i)); renderOosList();
      for (const q of Q.questions) {
        const s = saved.status[q.id];
        if (s?.state === "oos") markOos(q, s.scope, true);
        else if (s?.chosen) check(q, s.chosen);
      }
    }
  } catch (e) {}
}
</script>
</body>
</html>
~~~~

## `open.sh`
~~~~bash
#!/usr/bin/env bash
# Opens quiz.html in this folder as a focused app window (Chrome/Edge/Brave), otherwise in the default browser.
DIR="$(cd "$(dirname "$0")" && pwd)"
case "$(uname -s)" in
  Darwin)
    URL="file://$DIR/quiz.html"
    for app in "Google Chrome" "Microsoft Edge" "Brave Browser" "Chromium"; do
      if open -Ra "$app" 2>/dev/null; then
        open -na "$app" --args --app="$URL" --window-size=760,900; echo opened; exit 0
      fi
    done
    open "$DIR/quiz.html" ;;
  MINGW*|MSYS*|CYGWIN*)
    URL="file:///$(cygpath -m "$DIR/quiz.html")"
    for b in "/c/Program Files/Google/Chrome/Application/chrome.exe" \
             "/c/Program Files (x86)/Google/Chrome/Application/chrome.exe" \
             "$(cygpath -u "${LOCALAPPDATA:-C:/}")/Google/Chrome/Application/chrome.exe" \
             "/c/Program Files (x86)/Microsoft/Edge/Application/msedge.exe" \
             "/c/Program Files/Microsoft/Edge/Application/msedge.exe"; do
      if [ -x "$b" ]; then
        "$b" --app="$URL" --window-size=760,900 >/dev/null 2>&1 &
        echo opened; exit 0
      fi
    done
    cmd //c start "" "$URL" ;;
  *)
    URL="file://$DIR/quiz.html"
    for b in google-chrome chromium chromium-browser microsoft-edge brave-browser; do
      if command -v "$b" >/dev/null; then "$b" --app="$URL" >/dev/null 2>&1 & echo opened; exit 0; fi
    done
    xdg-open "$URL" ;;
esac
echo opened
~~~~

## `wait.sh`
~~~~bash
#!/usr/bin/env bash
# Usage: wait.sh <batch_id>
# Waits until the browser has downloaded quiz-<slug>-b<N>.json, moves it to results/, and prints "RESULT <path>".
DIR="$(cd "$(dirname "$0")" && pwd)"
N="$1"
SLUG="$(sed -n 's/.*"slug"[[:space:]]*:[[:space:]]*"\([^"]*\)".*/\1/p' "$DIR/config.json" | head -1)"
DL="$(sed -n 's/.*"downloads_dir"[[:space:]]*:[[:space:]]*"\([^"]*\)".*/\1/p' "$DIR/config.json" | head -1)"
DL="${DL:-$HOME/Downloads}"
DL="${DL/#\~/$HOME}"
while :; do
  f="$(ls -t "$DL/quiz-$SLUG-b$N.json" "$DL/quiz-$SLUG-b$N ("*").json" 2>/dev/null | head -1)"
  [ -n "$f" ] && break
  sleep 5
done
sleep 1   # let the browser finish writing
mkdir -p "$DIR/results"
mv "$f" "$DIR/results/b$N.json"
echo "RESULT $DIR/results/b$N.json"
~~~~
