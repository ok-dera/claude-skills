---
name: client-walkthrough
description: Turn a GitHub pull request into a client-facing PDF walkthrough with real screenshots captured via playwright-cli. Reads the PR, stages a disposable demo database with presentable sample data, runs the app locally, screenshots each user-facing change, and renders a polished A4 PDF report. Use when the user asks for a "client walkthrough", "release walkthrough", "PDF for the client", "show the client what's in this PR", or wants a non-technical document with screenshots covering a PR, release, or sprint of merged work.
---

# Client Walkthrough

Produce a client-facing A4 PDF that walks a non-technical reader through what a PR (or a
sprint of PRs) changed, with screenshots of the real running app on presentable demo data.

Work in a scratch directory with this layout:

```
scratch/
├── pr<N>.diff        # the PR, for reference while writing
├── shots/            # raw PNG screenshots, NN-slug.png
├── web/              # report.html + JPG copies of the shots
└── <Project>-Release-Walkthrough-<Name>.pdf
```

## 1. Read the PR and pick the sections

```bash
gh pr view <N> --json title,body,files
gh pr diff <N> > scratch/pr<N>.diff
```

List every **user-visible** change; ignore pure refactors, tests, CI. Group into 5–10
sections ordered by client interest (public-facing first, staff tooling after). For each
section decide the screenshots that prove it works — usually 1–3 per section, including
one alternate-state shot for anything with filters or toggles.

If the PR is large or it's unclear what the client cares about, confirm the section list
with the user before capturing anything.

## 2. Stage a disposable demo environment

Screenshots must show realistic, presentable data — never real user data, never obvious
QA junk ("test test", "QAUSER1").

- **Disposable DB**: copy the dev database under a new name (e.g. Postgres
  `createdb app_pdfdemo -T app_dev`) and point a tiny settings overlay at it:

  ```python
  # scratch/pdf_settings.py
  from config.dev import *
  DATABASES["default"]["NAME"] = "app_pdfdemo"
  EMAIL_BACKEND = "django.core.mail.backends.locmem.EmailBackend"
  ```

- **Curate, don't rebuild**: write `scratch/seed_demo.py` (run via `manage.py shell`) that
  renames existing QA records in place to realistic fictional names — deleting cascades
  away related data you want to show. Backfill whatever the new features need to look
  alive: varied dates, plausible dollar amounts, several rows per list view. Use clearly
  fictional but natural names ("Maya Ellison", not "Test User 3").
- Run the server against the overlay in the background:
  `python manage.py runserver 8000 --settings=pdf_settings` (adapt to the stack).

## 3. Capture screenshots with playwright-cli

```bash
playwright-cli open http://localhost:8000/
playwright-cli resize 1440 900          # consistent width for every shot
playwright-cli screenshot --filename=scratch/shots/01-landing-hero.png
playwright-cli screenshot e12 --filename=scratch/shots/08-chat-launcher-closeup.png  # element closeup
```

- Number files `NN-slug.png` in intended report order; insert late additions as `NNa-`.
- Use `snapshot` to find element refs, `fill`/`click` to log in as each role (public,
  customer, staff/admin) and to drive the app into each state before shooting.
- Full-page screenshots for long pages; element screenshots for closeups (a button, a
  form section). Capture both the default state and the state that shows the change
  (a past-month filter, a validation message, an under-18 signup).
- Re-shoot any frame with bad data rather than explaining it away in the caption.

**Capture gotchas** (each cost real time when this workflow was first run):

- **Scrolling via `eval` silently resets** — `window.scrollTo()` / `scrollIntoView()`
  report success but the page is back at `scrollY = 0` on the next command. Scroll with
  `mousemove <x> <y>` then `mousewheel 0 <delta>` instead, and verify position with
  `--raw eval "String(window.scrollY)"`. Downward deltas from a fresh `goto` are
  reliable; to go back up, re-navigate rather than sending a negative delta.
- **Forms on busy backgrounds**: element screenshots (`screenshot "<css selector>"`)
  beat viewport shots for legibility. If a container has no id, tag it first:
  `eval "document.querySelector(...).setAttribute('data-shot','1')"` then
  `screenshot "[data-shot='1']"`.
- **Styled toggles** (`peer sr-only` checkboxes) reject `check`/`click` on the input —
  click `label:has(#the-input)` instead.
- **Hover states leak into shots**: park the cursor in a page corner (`mousemove 1660 60`)
  before each screenshot or a table row renders highlighted.
- **One-time toasts** (post-login flashes): navigate once more before shooting.
- Server-side caches survive DB edits — Django's Site framework, for one — so restart the
  app server after changing config rows.

## 4. Convert shots to JPG

PNGs make a bloated PDF. Copy into `web/` as JPG (~80 quality):

```bash
for f in scratch/shots/*.png; do
  sips -s format jpeg -s formatOptions 80 "$f" --out "scratch/web/$(basename "${f%.png}").jpg"
done
# non-macOS: magick "$f" -quality 80 out.jpg
```

## 5. Write the report

Copy `assets/report-template.html` (bundled with this skill) to `scratch/web/report.html`
and fill it in. The template's print CSS (A4 `@page`, cover, TOC, numbered sections,
figure/caption, audience tags, before/after boxes) is proven — keep it, only swap the
`--accent` colors to the client's brand.

**Read [references/writing-style.md](references/writing-style.md) before writing any
prose.** The tone and vocabulary rules there are the difference between a client document
and a changelog.

Image srcs are bare filenames (`01-landing-hero.jpg`) since report.html sits next to them.

## 6. Render the PDF

playwright-cli refuses the `file:` protocol ("Access to \"file:\" protocol is blocked"),
so serve the report directory over local HTTP first:

```bash
(cd scratch/web && nohup python3 -m http.server 8765 --bind 127.0.0.1 >/dev/null 2>&1 &)
playwright-cli goto "http://127.0.0.1:8765/report.html"
# wait for images: eval a short setTimeout promise, then confirm
# [...document.images].filter(i=>!i.complete||i.naturalWidth===0).length === 0
playwright-cli pdf --filename="scratch/<Project>-Release-Walkthrough-<Name>.pdf"
pkill -f "http.server 8765"
```

## 7. QA pass

Open the PDF (the Read tool renders pages) and check: cover and TOC each fit one page, no
figure split across a page break, no screenshot showing junk data or a logged-in QA
username, captions match their images, file size is a few MB at most. Fix in report.html
or re-shoot, re-render, then hand the user the absolute PDF path.

Afterwards drop the demo database.
