# Client-facing writing style

The reader is a non-technical client or stakeholder. They paid for the work and want to
see what they got. They do not know what a model, endpoint, or migration is.

## Voice

- Plain, confident, slightly conversational British-report tone. "The holding page is gone."
  "Ambassadors could see how many people they had signed up, but not what that was worth."
- Lead every section with the outcome, not the mechanism. Heading = what changed for the
  user ("Guardian details are only asked of under-18s"), never the ticket or code change
  ("Refactor signup form validation").
- Name the old problem in one clause, then the new behaviour. A `.box.before-after`
  ("**Before:** … **Now:** …") does this well once per section; don't repeat it in prose too.
- Explain deliberate gaps rather than hiding them: "Lapsed and pending appear as dashes
  rather than as invented numbers … a deliberate choice to leave the gap visible."
- Honest, never salesy. No "exciting", "powerful", "seamless", "robust". No exclamation marks.

## Banned vocabulary

endpoint, model, migration, queryset, PR (except the cover's "pull request #N"), backend,
frontend, refactor, implement, deploy, merge, null, boolean, API (unless the feature IS an
API the client asked for). Translate: "the database" → "on record", "computed" →
"calculated", "admin" → "staff".

## Captions

Every screenshot gets a figcaption:
- Bold lead-in of 2–4 words naming what the figure is: `<b>The ledger.</b>`, `<b>A past month.</b>`
- Then 1–3 sentences telling the reader what to notice — the one detail that proves the
  feature works. Captions carry real information; never "Screenshot of the dashboard".

## Structure

- Cover: kicker, title ("What's new on {Project}"), one-line sub, prepared date + PR number,
  and the sample-data disclaimer note (keep it — clients ask whether names are real).
- TOC ("What's covered"): bold plain-words feature name + one grey sentence each.
- One `<section>` per feature, numbered eyebrow with an audience tag
  (`Everyone sees this` / `Staff only` / `{Role} sees this`).
- Order sections by what the client cares about most, not by commit order. Public-facing
  changes first, staff tooling after.
- 5–10 sections is right. Merge tiny fixes into one "smaller fixes" section or a `.box`
  inside a related section rather than giving each its own page.
