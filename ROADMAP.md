# SplitSlip — Roadmap

Single source of truth for **what is built, what is next, and which decisions are already locked**.
Read this first when picking up work or planning a feature.

> **For AI assistants:** this file tells you _what_ to build and _in what order_. _How_ to write code
> will live in each app's `AGENTS.md` (`api/`, `web/`, `ocr/`) once those apps exist. If this file
> disagrees with the code, the code wins — fix this file in the same change. Never re-open a
> decision listed under [Locked decisions](#locked-decisions) without the owner asking; propose,
> don't assume.

---

## Product

**SplitSlip** splits a restaurant bill between friends. Take a photo of the receipt, the app reads
the items with OCR, everyone taps what they ate, and the app works out who owes whom — service
charge, VAT and discounts spread in proportion. Built for the owner and their friends first.
Mobile-first web app.

Main flow:

1. Photo of the receipt → OCR → list of items, service charge, VAT, total.
2. Check and fix the list (OCR is never 100% right; the app flags a total that doesn't add up).
3. Add the people at the table.
4. Tap who had each item (shared items split evenly between the people picked).
5. Pick who paid the bill.
6. The app shows how much each person sends to the payer (with the payer's PromptPay QR).

## Repo layout

One git repo, independent apps side by side, the same way as `tang-pai-nai`. No monorepo tool and
no root `package.json`: each folder has its own dependencies, lint config, CI workflow and deploy.

| Path             | What                                | Stack                                                     |
| ---------------- | ----------------------------------- | --------------------------------------------------------- |
| `api/`           | REST API                            | Node + Express 5 + TypeScript, Drizzle, SQLite / Turso    |
| `web/`           | Mobile-first web app                | Angular (zoneless, signals), Tailwind v4, spartan/ui      |
| `ocr/`           | OCR model service                   | Python, OpenThai 2.0 OCR on a serverless GPU (Modal)      |
| `docs/features/` | Feature specs, shared by every app  | Markdown                                                  |
| `ROADMAP.md`     | This file                           |                                                           |

The web app never calls `ocr/` directly: `web → api → ocr`. Only `api/` holds the OCR API key.

---

## How we build a feature

**Vertical slices, one feature at a time.** Finish a feature end to end before starting the next.

**One feature (or one part of it, e.g. 5.2) = one new Claude session = one branch = one PR.** A new
session learns only from the repo (this file, the feature's spec, the `AGENTS.md` files), so a
decision agreed in chat but not written down is lost.

0. **Feature doc.** Write `docs/features/<feature>.md`, agree on it, commit it on its own
   (`docs(<feature>): ...`). It is the shared spec for API and web.
1. **API first.** Schema + migration, then the feature (entity → repository → service → controller
   / routes). Check it with `curl`.
2. **Web.** Screens under `web/src/app/features/<feature>/`, built from the form kit (#2).
3. **Tests.** Once the feature works end to end and the owner confirms the behaviour, write tests
   for both apps as the last commits of the PR. Not earlier: tests against code that is still
   changing just get rewritten.
4. **Update this file.** Move the feature to ✅ and note new decisions, deferred items and known
   issues for the next session.

Small, separate commits per step (Conventional Commits, scope prefixed `api` / `web` / `ocr` when
it helps, e.g. `feat(web/bill): ...`).

---

## Status

| #   | Feature / phase              | API | Web | OCR | Notes                                                                                     |
| --- | ---------------------------- | --- | --- | --- | ----------------------------------------------------------------------------------------- |
| 0   | Starter repos                | ⬜  | ⬜  | —   | `api/` from `node-express-ts-starter`; `web/` Angular + Tailwind + spartan; shared lint   |
| 1   | Local dev workflow           | ⬜  | ⬜  | —   | run both apps, `/api` dev proxy, `.env` setup, no real keys needed                        |
| 2   | Form kit + Storybook         | —   | ⬜  | —   | Reactive Forms fields on spartan/ui, each with a story. See below                         |
| 3   | Database foundation          | ⬜  | —   | —   | Drizzle + `@libsql/client`; local file DB, Turso in prod; migrations                      |
| 4   | Auth                         | ⬜  | ⬜  | —   | email code sign-in; reuse the approach from `tang-pai-nai`                                |
| 5   | Bill (typed by hand)         | ⬜  | ⬜  | —   | 5.1 bill + items + people; 5.2 who ate what; 5.3 split + who pays whom                    |
| 6   | OCR model trial              | —   | —   | ⬜  | Colab notebook: OpenThai OCR (Markdown vs Text) on 20+ real receipts, note accuracy       |
| 7   | OCR service                  | —   | —   | ⬜  | Modal endpoint, API-key check, image size limit; returns Markdown                         |
| 8   | Receipt scan                 | ⬜  | ⬜  | —   | upload photo → api → ocr → parse into items → review screen (fills the #5 bill)           |
| 9   | Settle up                    | ⬜  | ⬜  | —   | payer's PromptPay QR, share the result (image / LINE text)                                |
| 10  | History                      | ⬜  | ⬜  | —   | past bills, what each friend still owes across bills                                      |
| 11  | Deploy                       | ⬜  | ⬜  | ⬜  | Vercel ×2 (api, web), Turso, Modal; staging = `develop`, production = `main`              |

✅ done · 🚧 in progress · ⬜ not started · ⏸ deferred · — not applicable

Each row gets its `docs/features/<feature>.md` before work starts. **#5 comes before OCR on
purpose**: the app is useful with typed items, and OCR (#6–#8) only fills the same bill faster.

---

## #2 Form kit + Storybook

The owner builds Angular apps often and wants one set of form fields that work with Reactive Forms
and can be browsed in Storybook.

- Fields live in `web/src/app/shared/components/forms/<field>/` and implement
  `ControlValueAccessor`, so a feature writes `<app-input-field formControlName="name" />`.
- Built on the spartan/ui helm components in `web/src/app/shared/ui/` (generated, edited as little
  as possible).
- First set: `input-field`, `money-field` (THB, stored as satang), `textarea-field`,
  `select-field`, `checkbox-field`, `toggle-field`, `otp-field`.
- Each field shows label, hint, error (first validation error, translated), disabled and
  read-only, in a `<field>.stories.ts` with a Reactive Forms host so the story shows a real
  `FormControl`.
- Storybook runs with `npm run storybook` in `web/`.

## Locked decisions

- **Repo:** one repo, `api/` + `web/` + `ocr/` side by side, no root `package.json`.
- **Backend:** Node + Express + TypeScript for `api/` (same stack as `tang-pai-nai`, reuse its
  patterns and the starter). Python only in `ocr/`, because the model runs on Python
  (`transformers` / vLLM). Go isn't used.
- **Frontend:** Angular + spartan/ui + Tailwind.
- **Money:** whole satang (integers) everywhere; never floats. Rounding leftovers go to one
  person so the shares add up to the receipt total exactly.
- **Split rule:** each person's share of the food total × (receipt total ÷ food total). This
  covers service charge, VAT and discounts however the shop prints them.
- **OCR:** OpenThai 2.0 OCR (iApp) run by us on a serverless GPU, not a paid OCR API. The OCR
  service accepts only requests with our own secret key, set as a secret in Modal and an env var
  in `api/`. Never in the web app, never committed (this repo is public).
- **Git:** `develop` is the working branch; `main` gets merges from `develop` only, by the owner.
- **Language:** UI in Thai (default) and English. Code, comments, commits and docs in English.

## Open questions

- Markdown or Text variant of the OCR model — decided by the #6 trial.
- Parse OCR Markdown into items with our own code, or pass it to a small LLM for JSON — decided
  after #6 with real output.
- Where the split is computed: web only (pure function), or API too (to store the result). Lean:
  one pure function in web, API stores inputs only.
- Whether the form kit moves to its own repo / npm package for other Angular projects once it is
  stable.

## Backlog

- Several payers on one bill (one paid cash, one paid by card) with netting.
- Receipt images kept for later (storage, privacy).
- Split by quantity (one of 2× ชาเย็น each).
