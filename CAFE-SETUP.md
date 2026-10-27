# Aamcha Café — community café feature: setup, architecture & handover

Live routes
- Public page: https://asakelapahije.com/cafe.html
- Admin dashboard: https://asakelapahije.com/cafe-admin.html (noindex; server-side gated)

## 1. How it fits the existing site
The site is static HTML on GitHub Pages (repo `tejassalvi096/asa-kela-pahije`, branch `main`) with Firebase Firestore (`asa-kela-pahije`, asia-south1) as the only backend. No framework, no build step, no server. The café feature follows the same pattern:
- one self-contained page per route (inline CSS/JS, Firebase modular SDK 10.12.5 from gstatic),
- the same design tokens (paper/ink/yellow/red/teal, Bebas Neue + Rubik/Mukta, 3px ink borders, offset shadows),
- the same nav, with a new `☕ AAMCHA CAFÉ` link on index/fda/changle-lok, and a teal "NEW · COMMUNITY PROJECT" card on the home page,
- GA4 tag (G-ECGL08Y45G) kept consistent on the new pages.

New infrastructure that did not exist before: **Firebase Authentication** (Google sign-in + passwordless email link), used for identity on ideas, votes, interest, volunteers, applications and admin.

## 2. Files changed / added
| File | Change |
|---|---|
| `cafe.html` | NEW — public project page (tracker, PWYW explainer, participation, ideas + submission, voting rounds + results, interest registration, volunteer form, operator application, updates, budget vs actual, public ledger, FAQ) |
| `cafe-admin.html` | NEW — admin dashboard (ideas moderation, rounds, tally & publish, applications, interest/volunteers export, contributions & refunds, updates, budget & expenses, config, audit log) |
| `firestore.rules` | NEW in repo (deployed to Firebase) — full rule set incl. existing `janta`/`visits`/`noms` |
| `index.html` | nav link + café card |
| `fda.html`, `changle-lok.html` | nav link only |
| `CAFE-SETUP.md` | this document |

## 3. Data model (Firestore collections)
| Collection | Public read | Who writes | Notes |
|---|---|---|---|
| `admins/{uid}` | own doc only | console only | presence of doc = admin |
| `cafe_config/main` | yes | admin | target, acceptingMoney, policy, openNote, milestones[] |
| `cafe_ideas` | only statuses shortlisted/open/winner/planned/implemented/not_selected (+ owner, admin) | create: verified user, status forced `submitted`; update: admin | title, desc, problem, why, cost, link, name, uid, status, costEstimate, imageUrl, icon, considerations, adminNote, mergedInto |
| `cafe_ideas_private/{ideaId}` | admin | create: submitter | email |
| `cafe_rounds` | open/closed/published (admin sees drafts) | admin | title, desc, opensAt, closesAt, resultsAt, maxSelections, showInterim, ideaIds[], status, results{} |
| `cafe_votes/{roundId}_{uid}` | admin; owner; everyone when round.showInterim or round closed/published | create: verified user, **once per round, enforced by doc ID + rules**; no update/delete | roundId, uid, ideaIds[] |
| `cafe_interest` | admin | create: verified user (own uid/email) | name, email, amount, message |
| `cafe_interest_public` | yes | create: same user | amount only — feeds the tracker's "expressions of interest" count |
| `cafe_volunteers` | admin | create: verified user | |
| `cafe_applications` | admin + owner | create: verified user (status forced `submitted`); update: admin | operator applications |
| `cafe_contributions` | yes | admin | amount, date, status initiated/confirmed, method, displayName, contributorKey |
| `cafe_contributions_private/{id}` | admin | admin | email, reference/UTR |
| `cafe_refunds` | yes | admin | contributionId, amount, date, reason |
| `cafe_updates` | published only | admin | title, body, category, date, published |
| `cafe_budget` | yes | admin | planned lines |
| `cafe_expenses` / `cafe_expenses_private` | yes / admin | admin | actuals; receipt refs private |
| `cafe_audit` | admin | create: admin | every admin write: who, action, collection, doc, before/after |

Funding total = Σ confirmed contributions − Σ refunds, computed client-side from these records on every page load. Initiated payments, interest and pledges are displayed separately and never added to the total. There are no fabricated figures anywhere; empty states render until real records exist.

## 4. Security model
- Authorization is enforced by **Firestore security rules** (server-side). The page cannot grant itself anything.
- Admin = `admins/{uid}` exists. Add via Firebase console → Firestore → `admins` → document ID = the user's UID (shown on the admin page after sign-in if not yet an admin). Never writable from the client.
- Public visitors cannot write contributions, refunds, round results, idea statuses, updates, budget or config.
- Votes: doc ID must equal `{roundId}_{uid}`, uid must match the signed-in user, email must be verified, selection count ≤ round.maxSelections, no duplicates, only ideas in the round, round must be `open` and within opensAt/closesAt. Immutable after creation. This stops duplicate ballots per account; it does not stop one person using several email accounts — the FAQ says so plainly and results are advisory.
- Spam: sign-in required, honeypot field, client rate-limit (60 s), server length limits on every field, email must equal the signed-in token's email.
- PII (emails, phone, UTR, receipts) lives in admin-only collections/docs. Public ideas show the chosen display name only.
- Admins can delete records (spam, test data); every admin write/delete should be visible in `cafe_audit`. The audit log itself is append-only for everyone.
- Audit trail is written by the admin client on every write. Limitation: it is not tamper-proof without Cloud Functions (needs Blaze plan).

## 5. Environment / configuration
No secrets in code. The Firebase web config in the pages is a public identifier by design; access is controlled by rules + Auth.

Done in Firebase console during this build:
- Firestore rules published (copy in `firestore.rules`).
- Auth providers enabled: **Google**, **Email/Password with Email link (passwordless)**.
- Authorized domain added: `asakelapahije.com`.

Still to do by the owner:
- Sign in once on `cafe-admin.html`, copy the UID it shows, create `admins/{uid}` in Firestore (any field, e.g. `role: "owner"`).
- Optional: in Authentication → Templates, set a sender name for the email-link emails.

## 6. Deployment
Same as the rest of the site: upload changed files to the repo via GitHub (drag-drop or the Chrome upload flow used in this session); GitHub Pages deploys in ~1 minute. Rules are deployed separately in the Firebase console (Firestore → Rules → paste `firestore.rules` → Publish).

## 7. Admin workflow
1. Ideas tab → review `Submitted` ideas → set status (Under review / Shortlisted / Not selected with reason), add cost estimate, practical considerations, optional image URL, merge duplicates via "Merged into".
2. Rounds tab → create a round (title, dates, selections per person, interim visibility, results date, tick shortlisted ideas) → **Open round** (ideas become "Open for voting").
3. When closesAt passes (or you click Close) → **Tally & prepare results** → mark winners/decisions/reasons → **Publish results**. Idea statuses update (Winner / Not selected / back to Shortlisted). The public page shows ranking, total ballots, cost, decision and reasons.
4. Contributions tab → record each receipt as it lands in the account (Confirmed) or Initiated; mark Confirmed later. Refunds reference a contribution ID. Tracker updates on next page load.
5. Updates / Budget / Config tabs as needed. Everything is logged in Audit.

## 8. Tests performed (27 Oct 2026, live site, then purged)
| # | Journey | Result |
|---|---|---|
| 1 | Visitor opens cafe.html (mobile 390px + desktop) | renders, no JS errors, empty states shown |
| 2 | Visitor submits an idea (signed in via Google) | 3 ideas saved as `submitted`; email stored privately |
| 3 | Admin reviews & shortlists | 3 ideas → Shortlisted with cost estimate, considerations, icon |
| 4 | Admin opens a voting round | round `open`, ideas flipped to "Open for voting", visible publicly |
| 5 | Visitor votes within rules | 2 of max 2 selected; 3rd selection blocked client-side; ballot saved |
| 6 | Duplicate / manipulated votes | rejected by rules: same doc ID again, second ballot with other ID, random ID, over-limit, delete own vote, update own vote — all `permission-denied`; ballot count stayed 1 |
| 7 | Results calculated | tally via server count queries: 1 ballot, 1/1/0 |
| 8 | Admin publishes outcome | ranking, decisions, reasons, cost shown publicly; winners → `winner`, declined → `not_selected` with reason |
| 9 | Visitor registers interest (₹500) | counted as "1 people · ₹500 indicated"; raised stayed ₹0 |
| 10 | Visitor submits operator application + volunteer offer | both saved; visible to admin only |
| 11 | Admin publishes a project update | shown in Updates + "Latest update" box |
| 12 | Funding totals from records | ₹1,000 confirmed + ₹500 initiated → raised ₹1,000, initiated shown separately |
| 13 | Refund | ₹300 refund → net ₹700 on hero, tracker and public ledger |
| 14 | Unauthorised access | anonymous client: writes to contributions/config/ideas/votes and reads of interest/applications/submitted ideas/audit all `permission-denied`; public reads OK; non-admin sign-in sees the gate, not the dashboard |
All test records were deleted afterwards (logged as `purge_test_data` in `cafe_audit`).

## 9. Known limitations
- No payment gateway. `acceptingMoney` only flips the label; contributions are recorded manually by an admin from bank/UPI statements. A gateway (Razorpay/Instamojo) would need a server/Cloud Function to verify webhooks — not possible on the free Spark plan.
- No image upload; ideas accept an https image URL instead (admins can set one after review).
- Idea-submission "confirmation email" is the sign-in itself (Google or the email link). No separate transactional email is sent — that needs a mail service.
- Audit log is client-written by admins (trustworthy against the public, not against a rogue admin).
- Duplicate-vote prevention is per verified account, not per human.
- All totals are computed in the browser from public ledger documents; fine at this scale, would need aggregation docs beyond a few thousand records.

## 9a. Model change (27 Oct, evening) — contribution-first, merit-selected operator
- Pay-what-you-wish removed. Minimum contribution ₹100, **no refunds** (stated in hero, terms box, FAQ, footer).
- One contributor is selected to operate the café **on merit**: contribute → apply → panel review → presentation day → partnership agreement with the AKP entity (operates, does not own). Never random, never for sale — this keeps it a skill competition, not a lottery (Lotteries (Regulation) Act / Maharashtra ban; Razorpay prohibits prize/chance schemes).
- "The math" section: 40 lakh views × ₹100 arithmetic, explicitly labelled a what-if, with the live real number beside it and an interactive calculator.
- Payments: Razorpay **Payment Page** (hosted; no server needed). The Pay button only renders when config has `acceptingMoney=true`, `entityName`, `failurePolicy` and a valid `razorpayPageUrl` (pages.razorpay.com / rzp.io). Admin imports Razorpay's payment CSV (`captured` only, idempotent by payment id) so the tracker shows only money that landed.
- Operator application now records the contribution email (`payEmail`) for matching.

## 10. Needs your approval before real fundraising opens
1. Legal entity that receives and holds the funds (and its bank account).
2. Permitted use of funds.
3. Campaign terms (gift/contribution nature; no equity/returns/ownership).
4. Refund and cancellation rules.
5. What happens if the café cannot proceed.
6. Surplus handling.
7. Accounting and tax treatment (and whether any 80G claim is permitted — currently the page says none is claimed).
8. Any non-financial thank-yous (e.g. name wall) — text to be reviewed first.
9. Operator selection criteria and process (to be published before any selection).
10. Payment-gateway choice and the server component to verify it.
Until these are published, keep `Accepting money = No` (pre-launch) — the page then only registers interest.
