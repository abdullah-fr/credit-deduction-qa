# credit-deduction-qa

A Playwright-based QA automation tool that verifies whether SaaS tools correctly deduct
credits/quota from a user's account when a conversion or generation action is run.

Built to test a portfolio of 10 content/OCR/writing-tool websites that share a common
underlying billing backend, across staging and live environments.

## What it does

For every tool on a site, the script:

1. Reads the account's current credit balance.
2. Runs the tool (uploads a test file, or types sample text where the tool has no
   file input) and clicks its convert/submit button.
3. Waits for the tool to finish, then re-reads the credit balance.
4. Flags **PASS** if the balance changed, **FAIL** if it didn't — and on sites that expose
   an exact per-tool credit rate, can additionally verify the *exact* amount deducted.

It runs against a **real logged-in browser session** (via Chrome DevTools Protocol) rather
than a fresh automated profile, so it exercises the actual account state a real user would
see — no test-account cookies or mocked auth.

## Why this isn't a simple "click a button and check a number" script

Every one of the 10 target sites renders its UI differently, and several behaviors that
looked reliable on one site turned out to be wrong on another. The script's design reflects
the actual failure modes hit while building it:

- **No single "credit balance" format.** Some sites show a `used / total` ratio in one
  element; others (e.g. a "Plan Details" table) split the same info across
  `Credits Allowed` / `Credits Used` columns; others show a plain `"<N> used"` string next
  to an `Unlimited` allowance that must be ignored. The balance reader tries several
  extraction strategies per site rather than assuming one layout.
- **Multiple independent credit pools.** Some accounts show more than one pool at once
  (e.g. a "Monthly" plan and a "Premium" plan simultaneously, or per-tool-category pools
  like "Plagiarism Checker" vs "AI Writing Tools"). The script isolates and checks the
  *specific* pool a given tool actually draws from, instead of treating the whole page as
  one balance and getting confused when only one of several numbers moves.
- **Convert buttons aren't consistently `<button>` elements.** Some sites use a styled
  `<div>`/`<span>` with a click handler instead. Selector matching falls back through
  button → div/span → link → `[role=button]`, guarded so it never blind-clicks an
  ambiguous match (e.g. a site-wide nav link that happens to also say "Convert").
- **"Reload the balance" isn't always a real reload.** Navigating to the exact same
  account-page URL twice in a row can leave a single-page app showing stale, cached
  numbers until a real (cache-busted) navigation is forced.
- **Some tools take typed/pasted text, not a file upload.** A few tools have no upload
  control at all — sample text is typed into the relevant textarea/contenteditable
  element instead.

Each of these is handled through a small number of general-purpose, config-driven
mechanisms (see **Architecture** below) rather than one-off hacks per site, so adding a
new site or tool is mostly a matter of adding data, not new control flow.

## Architecture

| Concern | Mechanism |
|---|---|
| Finding a submit/convert control | `find_clickable()` — tries a prioritized candidate-selector list per (site, tool), skipping any match that's ambiguous (matches >1 element) rather than guessing |
| Reading the credit balance | `get_credit_balance()` — generic whole-page extraction by default; `get_credit_balance_for_pool()` for sites with multiple/separately-labeled pools |
| Per-tool selector/behavior overrides | Small lookup dicts keyed by `(domain, tool_name)` — `SUBMIT_BTN_OVERRIDES`, `RESULT_INDICATOR_OVERRIDES`, `TEXT_INPUT_OVERRIDES` — checked ahead of generic guessing, with a `"default"` fallback per site |
| Detecting a finished conversion | Two-tier selector list: high-confidence markers (a real "Start Over"/download control, which can only exist once a result exists) checked first, weaker guessed markers as a fallback |
| Stale/cached account pages | `cache_busted_url()` appends a changing query param so every balance check is a genuinely fresh navigation |
| Debuggability | `dump_debug_info()` saves a screenshot + relevant HTML snippet whenever a selector fails to match, so a fix can be written from real markup instead of guessing blind |

## Supported sites

editpad.org · grammarcheck.ai · imagetotext.cc · imagetotext.info · imagetotext.io ·
jpgtotext.com · ocr.best · paraphrasing.io · prepostseo.com · summarizer.org

Per-site tool lists, resolved tool URLs, and per-tool credit rates are seeded from
`credits_overview_data.txt` / `resolved_tool_urls.json` and cached to `site_mappings.json`
on first run.

## Requirements

- Python 3.12
- Playwright for Python (`pip install playwright && playwright install`)
- A Chromium-based browser (Brave/Chrome/Edge) — the script launches it with
  `--remote-debugging-port=9222` if it isn't already running that way, so it can attach to
  your real logged-in session rather than a blank automated profile

## Usage

```
python credits_deduction.py
```

You'll be prompted to pick an environment (Staging/Live), then a site (or "Run All Sites").
The script opens an account tab and a tool tab, and walks through each of that site's tools.

## Project structure

```
credits_deduction.py        # main script
site_mappings.json           # cached per-site tool list + resolved URLs (auto-generated/merged)
credits_overview_data.txt    # seed: per-site tool → credit rate
resolved_tool_urls.json      # seed: per-site tool → confirmed URL path
Test Data/                   # sample upload files, organized by type (image/, pdf/, word/, txt/, ...)
debug/                       # auto-saved screenshots + HTML snippets on selector failures
```

## Known limitations

- Selector overrides for some tools (e.g. grammarcheck.ai's text-input box) are
  best-effort candidate lists rather than confirmed-exact selectors, and may need
  tightening from a `debug/` dump on first run against a new site.
- Exact-amount credit verification (vs. deduction-only) is currently only implemented
  for sites that expose a documented per-tool credit rate.

## Status

Personal QA tooling built and iterated against real staging/live sites as part of a
broader automation portfolio. Not affiliated with or endorsed by the tested sites.
