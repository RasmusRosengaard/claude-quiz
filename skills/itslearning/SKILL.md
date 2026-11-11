---
name: itslearning
description: Henter undervisningsmateriale og lektier (forberedelse) fra itslearning-planer for valgte fag via Claude in Chrome (brugerens egen login), gemmer det lokalt, eventuelt i en mappe på skrivebordet med tomme pladsholdere for bogkapitler, og kobler det til /quiz, så spørgsmålene laves ud fra materialet og lektierne i relation til læringsmålene. Use when the user asks to fetch/import material, plans or homework from itslearning, or to build a quiz from itslearning.
argument-hint: "[itslearning-URL] [quiz-fag]"
---

# itslearning → material → /quiz

Fetch everything a course's plans contain (files, slides, links, homework/preparation texts) for the subjects the user picks, and hand it over to `/quiz`. Runs in the user's own Chrome with their own login via **Claude in Chrome**. Never ask for passwords, and never enter login details. If the user isn't logged in: ask them to log in themselves in the opened tab, and wait.

Before the first browser step: load the `chrome-browser` skill (or `claude-in-chrome`) and follow its rules (tabs_context first, own tab, close it afterwards, stop after 2–3 failed attempts).

## Privacy rules (important)
- The signed links to `platform.itslearning.com`/`resource.itslearning.com` contain the user's name, e-mail, user-id and a signature. NEVER print them in tool output or chat, and never save them to files. Only print host + path or ids. Navigate to them via `location.href = src` inside the JavaScript tool, without returning the URL.
- Material is downloaded for the user's own use and stays local. Never commit or push it anywhere.

## 1. Settings (ask with AskUserQuestion; skip what the arguments answer)
1. **itslearning address**, e.g. `https://ucl.itslearning.com`. Default: the last one in `~/tools/quiz/itslearning.json`.
2. **Group/course**: once logged in, fetch `GET /restapi/personal/courses/v1?PageIndex=0&PageSize=100` and show the course names (e.g. "SWOF26 + SWVF26"). Remember `courseId`.
3. **Which plan tab(s)**: "Tidligere" (`past`), "Aktuelle" (`current`), or both.
4. **Subjects**: fetch the plans (step 2) and group by `topic.name`. Show each subject with its number of plans (multiSelect), e.g. "Systemintegration - efterår 2026 (13)", "Store Systemer (7)".
5. **Quiz subject**: which `/quiz` slug the material should be linked to (existing or new, e.g. `eip`). Default: derived from the subject name.
6. **Folder on the desktop?** "Ja, læg materialet på skrivebordet" / "Nej, kun i quiz-mappen".
Save the choices in `~/tools/quiz/itslearning.json` (`{base, courseId, courseName, tabs, topics, slug, desktop}`), so a later run can just ask "Same as last time?".

## 2. Fetch the plan list (JSON API, no clicking)
In the itslearning tab (same-origin `fetch` with the user's cookies):
`GET {base}/restapi/personal/course/{courseId}/plans/{past|current}/v1?PageIndex=0&PageSize=100&ChildId=0`
→ `{entityArray, total}`. If `total > 100`, fetch more pages. Per plan: `id`, `title`, `start`, `stop`, `topic.name`, `description` (HTML = **"Forberedelse og pædagogisk tilrettelæggelse"**, i.e. the homework text), `elements[]` with `id`, `title`, `type`, `typeName`, `planPageUrl`.
Filter on the chosen `topic.name`. Sort by `start`.
Only `GET` against the API. NEVER call `PUT`/`POST` endpoints, e.g. `/progress/completion/v1`. Note: opening plan pages in the normal UI can mark items as "set" (seen). That's itslearning's own behaviour, so prefer the API where possible.

## 3. Per plan: homework (the most important part)
Collect into `lektier.md` (one section per plan, in date order):
- `## <date> – <plan title>` + `description` stripped of HTML. This is the homework text, e.g. "Messaging Channels (chap. 4)".
- Elements of type **Arbejdsopgave** (`CustomActivity`), especially ones named "Forberedelse"/"Lektie": open `{base}{planPageUrl}` and read the text via JavaScript from the same-origin iframe: `iframe.contentDocument.body.innerText` (wait ~2 s for it to load). Take the "Beskrivelse" part, e.g. "Read EIP kapitel 4; Handin RabbitMQ task…".
- **Afleveringsopgave**: open it the same way and save the assignment text (it often shows what is expected).
- **Book references** in the homework text (e.g. "EIP kapitel 4", "chap. 4", "Newman ch. 3"): resolve the abbreviation to the book title + author (EIP = Enterprise Integration Patterns, Hohpe & Woolf; "Building Microservices" = Sam Newman, etc.) and the chapter title, ONLY if you're sure. Record them under `### Bøger` in the section and in `config.materials` as `{"type": "book", "ref": "<Titel – Forfatter>", "label": "<Bog> kap. <N>: <kapiteltitel>"}`.

## 4. Per plan: files and links
For each element (keep the plan order). Element id = `LearningToolElementId`.
- **Fil** (`typeName` "Fil" / "Microsoft Word" / "PowerPoint" …):
  1. `fetch('/LearningToolElement/ViewLearningToolElement.aspx?LearningToolElementId=<id>')` → find `iframe[src*="platform.itslearning.com"]` in the HTML → `location.href = src` (do NOT return it).
  2. Wait ~3 s. The page is now `resource.itslearning.com/File/View.aspx`. Click the link with the text **"Hent"** via JS (`[...document.querySelectorAll('a')].find(a => a.textContent.trim() === 'Hent').click()`). The browser saves the file in Downloads under its original name.
  3. Navigate back to `{base}` before the next element (the API calls require the itslearning domain).
- **Link** (`learningToolId` 5010): same SSO navigation. The page is `resource.itslearning.com/Link/View.aspx`, and the target is the first external `a[href]`. Save `title + URL` in `links.md` (the URL itself is fine to save; it's not signed). If the target is `docs.google.com/presentation/d/<ID>` or `docs.google.com/document/d/<ID>`: download it as PDF by navigating to `https://docs.google.com/presentation/d/<ID>/export/pdf` (document: `/document/d/<ID>/export?format=pdf`). If it requires Google access the user doesn't have, just note it in `links.md`. YouTube and other sites: only note the link.
- **Side**: open `planPageUrl`, read the iframe text, and save it as `<title>.md`.
- **Mappe**: open `planPageUrl` and process the elements inside the folder the same way (one level). If it is called "old stuff" or similar, ask before fetching.
- Wait for each download: in bash, check for a new file in Downloads (`ls -t`), max ~30 s. Skip `.crdownload`/`.download` files.
- Move the downloaded file from Downloads into the plan's folder (see 5). For name collisions: add ` (2)`.
- Skip `.zip` files larger than 50 MB and video files, but list them in `links.md`.

## 5. Folder structure
Root: if `desktop = true` → `<Desktop>/<Fagnavn> – materiale/`, otherwise `~/tools/quiz/<slug>/materiale/`.
Find the desktop (it may be in OneDrive): Windows `powershell -NoProfile -Command "[Environment]::GetFolderPath('Desktop')"` (convert with `cygpath -u`), macOS `~/Desktop`.
```
<root>/
  lektier.md                      ← all homework/preparation in date order (the core of the quiz)
  links.md                        ← all links, per plan
  Bøger/
    Enterprise Integration Patterns – kap. 4 Messaging Channels.txt   ← EMPTY placeholder file
  <topic>/
    2026-09-14 – ODENSE - Mdam - dag 4/
      Slides DAY 4 - Messaging Channels.pdf
      KombineretEksamensSpørgsmål.pdf
      Forberedelse.md
```
- File and folder names: remove `\ / : * ? " < > |` and trim to max 80 characters.
- **Books:** for every book chapter referenced, create an EMPTY file `Bøger/<Bogtitel> – kap. <N> <kapiteltitel>.txt` (without the chapter title if you're not sure of it). It's a placeholder for the user to fill in themselves (e.g. with their own notes or a scanned chapter). Never fetch book content from the internet.
- Write `<root>/LÆS MIG.txt` with: when it was fetched, from which course/subjects, and that the folder can be re-run with the itslearning skill (existing files are skipped).
- Rerun: skip elements whose file already exists (match on name), so only new material is fetched.

## 6. Link it to /quiz
1. If `~/tools/quiz/<slug>/` doesn't exist: tell the user to run `/quiz <slug>` (it asks for learning goals and settings), then come back. Or ask for the learning goals now and create the subject per the /quiz conventions.
2. Add to `~/tools/quiz/<slug>/config.json` `materials`:
   - `{"type": "folder", "ref": "<root>", "label": "itslearning: <fag>", "digested": false}`
   - the book entries from step 3
   and add to `wishes`: `"Prioritér lektierne i lektier.md: hvert lektie-emne skal dækkes af spørgsmål, koblet til læringsmålene i goals.md"`.
3. Digest the material now (as the Materials step in /quiz describes): read `lektier.md` FIRST in full, then the files (pdf: Read with `pages`; pptx/docx: unzip + strip XML). Write `materials.md` with one section per plan, `## <date> – <plan> [LEKTIE: …]`, containing the key concepts, definitions and examples, and for each section which learning goal(s) from `goals.md` it supports. Then set `digested: true`.
4. Finish with a short overview: number of plans, files, links and book chapters, what was skipped and why, the folder location, and that `/quiz <slug>` now makes questions from the material and homework.

## Fallbacks
- Not logged in, MitID/SSO login, or a captcha: the user does it themselves in the tab. Wait and ask.
- The API returns 401/403: the session has expired. Ask the user to reload itslearning.
- The layout or API has changed (404 on the endpoints): go through the UI instead (Planer → the tab → plan → elements) with read_page/find. Tell the user that the skill should be updated.
