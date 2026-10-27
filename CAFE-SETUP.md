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

## 8. Tests performed
See the test log at the end of the session summary (items 1–14 of the brief). Local Playwright render test on mobile (390px) and desktop: no JS errors, all sections render, empty states shown, auth gate opens on protected actions.

## 9. Known limitations
- No payment gateway. `acceptingMoney` only flips the label; contributions are recorded manually by an admin from bank/UPI statements. A gateway (Razorpay/Instamojo) would need a server/Cloud Function to verify webhooks — not possible on the free Spark plan.
- No image upload; ideas accept an https image URL instead (admins can set one after review).
- Idea-submission "confirmation email" is the sign-in itself (Google or the email link). No separate transactional email is sent — that needs a mail service.
- Audit log is client-written by admins (trustworthy against the public, not against a rogue admin).
- Duplicate-vote prevention is per verified account, not per human.
- All totals are computed in the browser from public ledger documents; fine at this scale, would need aggregation docs beyond a few thousand records.

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
