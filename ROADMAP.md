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
charge, VAT and discounts spread in proportion. Mobile-first web app, **open to anyone with no
sign-in**: bills are kept on the device, and the API only reads receipts.

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

| Path             | What                               | Stack                                                       |
| ---------------- | ---------------------------------- | ----------------------------------------------------------- |
| `api/`           | OCR gateway: rate limits, forwards | Node + Express 5 + TypeScript, Upstash Redis for the limits |
| `web/`           | Mobile-first web app               | Angular (zoneless, signals), Tailwind v4, spartan/ui        |
| `ocr/`           | OCR model service                  | Python, OpenThai 2.0 OCR on a serverless GPU (Modal)        |
| `docs/features/` | Feature specs, shared by every app | Markdown                                                    |
| `ROADMAP.md`     | This file                          |                                                             |

Form fields come from **[`ng-form-kit`](https://github.com/12ain13owz/ng-form-kit)**, the owner's
own Angular library in its own repo (see [Form kit](#form-kit-ng-form-kit)). `web/` installs it as
a package; it is not built here.

The web app never calls `ocr/` directly: `web → api → ocr`. Only `api/` holds the OCR API key.

---

## How we build a feature

**Vertical slices, one feature at a time.** Finish a feature end to end before starting the next.

**One feature (or one part of it, e.g. 3.2) = one new Claude session = one branch = one PR.** A new
session learns only from the repo (this file, the feature's spec, the `AGENTS.md` files), so a
decision agreed in chat but not written down is lost.

0. **Feature doc.** Write `docs/features/<feature>.md`, agree on it, commit it on its own
   (`docs(<feature>): ...`). It is the shared spec for every app it touches.
1. **API first** (when the feature has an API part). Check it with `curl`.
2. **Web.** Screens under `web/src/app/features/<feature>/`, built from `ng-form-kit` fields.
3. **Tests.** Once the feature works end to end and the owner confirms the behaviour, write tests
   as the last commits of the PR. Not earlier: tests against code that is still changing just get
   rewritten.
4. **Update this file.** Move the feature to ✅ and note new decisions, deferred items and known
   issues for the next session.

Small, separate commits per step (Conventional Commits, scope prefixed `api` / `web` / `ocr` when
it helps, e.g. `feat(web/bill): ...`).

---

## Status

| #   | Feature / phase       | API | Web | OCR | Notes                                                                                        |
| --- | --------------------- | --- | --- | --- | -------------------------------------------------------------------------------------------- |
| 0   | Starter repos         | ⬜  | ⬜  | —   | `api/` from `node-express-ts-starter`; `web/` Angular + Tailwind + spartan; shared lint      |
| 1   | Local dev workflow    | ⬜  | ⬜  | —   | run both apps, `/api` dev proxy, `.env` setup, no real keys needed                           |
| 2   | Form kit hookup       | —   | ⬜  | —   | install `ng-form-kit`, Tailwind `@source` for it; waits for the kit's first release          |
| 3   | Bill (on the device)  | —   | ⬜  | —   | 3.1 items + people, saved on the device; 3.2 who ate what; 3.3 split + who pays whom         |
| 4   | OCR model trial       | —   | —   | ⬜  | Colab notebook: OpenThai OCR (Markdown vs Text) on 20+ real receipts, note accuracy          |
| 5   | OCR service           | —   | —   | ⬜  | Modal endpoint, API-key check, max 1 container, image size limit; returns Markdown           |
| 6   | Receipt scan          | ⬜  | ⬜  | —   | photo → api → ocr → items → review screen (fills the #3 bill); image never stored            |
| 7   | Abuse protection      | ⬜  | ⬜  | ⬜  | per-IP and daily limits, Cloudflare Turnstile, size / type checks. See below                 |
| 8   | Settle up             | —   | ⬜  | —   | payer's PromptPay QR, share the result (image / LINE text)                                   |
| 9   | History               | —   | ⬜  | —   | past bills on the device, what each friend still owes across bills                           |
| 10  | Deploy                | ⬜  | ⬜  | ⬜  | Vercel ×2 (api, web), Upstash, Modal; staging = `develop`, production = `main`               |

✅ done · 🚧 in progress · ⬜ not started · ⏸ deferred · — not applicable

Each row gets its `docs/features/<feature>.md` before work starts. **The bill (#3) comes before
OCR on purpose**: the app is useful with typed items, and OCR (#4–#6) only fills the same bill
faster. **#7 must be done before #10**: the scan endpoint is public and every call costs GPU time.

---

## Form kit (`ng-form-kit`)

The owner builds Angular apps often, so the form fields live in their own repo,
`12ain13owz/ng-form-kit`, as an Angular library with Storybook. That repo has its own roadmap; this
section only records what SplitSlip expects from it.

- Package `@12ain13owz/ng-form-kit` on **GitHub Packages** (npm public later, if ever). Built
  with `ng-packagr`; Storybook is the kit's own playground and docs.
- Every field implements `ControlValueAccessor`, so a feature writes
  `<nfk-input-field formControlName="name" />` with Reactive Forms.
- Built on spartan/ui: the helm code is copied into the kit; `@angular/*` and
  `@spartan-ng/brain` are **peer dependencies**, so an app never carries two copies.
- Apps that install it add `@source "../node_modules/@12ain13owz/ng-form-kit";` to their Tailwind
  CSS, or the kit's classes aren't generated.
- Fields SplitSlip needs first: `input-field`, `money-field` (THB, value in satang),
  `select-field`, `checkbox-field`.
- While both change often, `web/` may point at the kit's source (`npm link` or a path alias) in
  local dev only; commits always use a released version.

## Abuse protection (#7)

No sign-in means anyone can call the scan endpoint, and each call runs a GPU. Layers, all of them:

| Layer           | Rule (starting values, tune later)                       | Why                                      |
| --------------- | -------------------------------------------------------- | ---------------------------------------- |
| Per IP          | 10 scans / hour                                          | one person hammering the button          |
| Whole app       | 300 scans / day, then scanning pauses until tomorrow     | caps the bill even across many IPs       |
| GPU             | Modal: at most 1 container at a time                     | a hard ceiling on spend                  |
| Bots            | Cloudflare Turnstile token checked by `api/`             | scripts can't call the API directly      |
| Image           | ≤ 5 MB, jpg / png / webp only; the web app resizes first | less GPU time per call                   |

Counters live in **Upstash Redis**: in-memory counters (`express-rate-limit`'s default store) don't
work on Vercel, where each serverless instance counts on its own. When a limit is hit, the web app
says so and the bill can still be typed by hand.

## Locked decisions

- **Repo:** one repo, `api/` + `web/` + `ocr/` side by side, no root `package.json`.
- **No sign-in, no database (for now):** bills, people and history are kept on the device
  (IndexedDB). The API keeps no user data. Auth and a server database are in the backlog.
- **Backend:** Node + Express + TypeScript for `api/` (same stack as `tang-pai-nai`, reuse its
  patterns and the starter). Python only in `ocr/`, because the model runs on Python
  (`transformers` / vLLM). Go isn't used.
- **Frontend:** Angular + spartan/ui + Tailwind; form fields from `ng-form-kit`, not written here.
- **Receipt images are never stored:** read, then dropped, on both `api/` and `ocr/`.
- **Money:** whole satang (integers) everywhere; never floats. Rounding leftovers go to one
  person so the shares add up to the receipt total exactly.
- **Split rule:** each person's share of the food total × (receipt total ÷ food total). This
  covers service charge, VAT and discounts however the shop prints them. Computed in `web/` as a
  pure function.
- **OCR:** OpenThai 2.0 OCR (iApp) run by us on a serverless GPU, not a paid OCR API. The OCR
  service accepts only requests with our own secret key, set as a secret in Modal and an env var
  in `api/`. Never in the web app, never committed (this repo is public).
- **Git:** `develop` is the working branch; `main` gets merges from `develop` only, by the owner.
- **Language:** UI in Thai (default) and English. Code, comments, commits and docs in English.

## Open questions

- Markdown or Text variant of the OCR model — decided by the #4 trial.
- Parse OCR Markdown into items with our own code, or pass it to a small LLM for JSON — decided
  after #4 with real output.
- Exact limit values in #7 — start with the table above, tune from real use.

## Backlog

- Sign-in and a server database: bills shared between friends' phones, history across devices.
- Several payers on one bill (one paid cash, one paid by card) with netting.
- Split by quantity (one of 2× ชาเย็น each).
