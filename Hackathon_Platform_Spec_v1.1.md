# Hackathon Platform Spec v1.1

Oct 14, 2026 · @Umar · supersedes v1

## Changes from v1

| Area | v1 | v1.1 |
| --- | --- | --- |
| Judge login | Google login or admin-generated password | Admin-generated password only; no social login, no sign-up route |
| Categories | Teams self-nominate up to 3; one award per category | **Challenges** of two kinds: 9 open (every project eligible, no nomination) and 4 pitched (teams opt in, up to 3) |
| Hardware | Not modelled | `has_hardware` checkbox on the submission; filters the Best Hardware Hack nominee list |
| Deliberation | Per-category nominees sorted by overall rank | Per-award nominee lists sorted by the matching rubric criterion, using per-judge, per-criterion normalization |
| Multiple awards | A team can win multiple awards | Admin setting: allow, warn (default) or block; venture award exempt by default |
| Pitching | 10-min slot, single room implied | 10-min default slot, coordinator flexes live; single room locked, with rationale |
| File storage | Railway storage "if offered", else R2 | Railway Bucket (verified: private, S3-compatible); uploads go through the app server; served by short-lived presigned GET URLs |
| Updates page | "Updates" | Renamed **Announcements** (same behaviour) |
| ORM | Drizzle or Prisma | Drizzle |
| Close-time enforcement | Trigger on `submission` | Triggers on `submission` and every child table |
| Timeline | Finish one week before the event | Code complete Oct 20, dry run Oct 21, freeze Oct 23 |

## Overview

We build one small custom web app to run a private, in-person, two-day hackathon end to end: event info, project submissions, live judging and awards. Hosted platforms were evaluated and set aside in favour of ownership and reusable tooling.

| Parameter | Value |
| --- | --- |
| Event | MC² Princeton Summit hackathon, Oct 24–25, 2026, MC² Works + Labs, East Windsor, NJ |
| Timezone | America/New_York for every configured time |
| Format | In person, 2 days, pitches on day 2 |
| Participants | 30–40, mostly young professionals and some high schoolers; a few college students |
| Teams | \~8–13, 1–4 members, self-formed in the room |
| Judges | \~6, all judges see every pitch in one room |
| Challenges | 13: 9 open, 4 pitched (see Challenges) |
| Judging model | Devfolio/ETH-style rubric feeds ranked lists; judges deliberate final awards |
| Event conduct model | Devpost-style phases, pages and submission format |

Goals:

- Frictionless for participants: no accounts, a QR code and an access code get them in.
- Easy for judges: phone-friendly scoring that follows the live pitch.
- Trustworthy results: hard submission deadline, private per-judge scores, normalized rankings.
- Legal, NDAs and consent are handled outside this project. Open item for legal: if any participant is under 13, collecting their email may require parental consent; the form label can ask for a parent's email in that case.

## Locked decisions

Everything below was agreed in planning and is not reopened without a reason.

| Area | Decision |
| --- | --- |
| Build vs buy | Custom app; HackHQ ($199/event) was the buy alternative |
| App shape | One Next.js app; `/judge` and `/admin` routes behind auth |
| Participant identity | No accounts, no team management; team details captured on the submission |
| Outsider protection | QR code carries an event access code; per-team private edit link |
| Judge identity | Admin-created staff with admin-generated passwords (shown once). No social login, no public sign-up |
| Pitch coordination | "Now pitching" control driven by a coordinator (MC); admin as backup |
| Pitch room | One room; every judge sees every pitch (rationale below) |
| Pitch slot | 10 minutes by default; the coordinator flexes it live; the app does not enforce timing |
| Rubric | Devfolio criteria, 0–10 each, equal weights by default |
| Results | Per-judge normalized scores feed deliberation; judges decide awards; shared places allowed |
| Challenges | Open challenges: every submitted project is eligible, no nomination. Pitched challenges: teams opt in to 0–3 and must address them in the pitch |
| Multiple awards | Admin setting (allow / warn / block), default warn |
| AI tools | Fully allowed, no disclosure field |
| Repos | Public repos expected; zip is not required |
| Screenshots | Up to 5 per submission, uploaded through the app server to a private Railway Bucket |
| Announcements | Organizer-posted, newest first, on a public page |
| Contact data | Kept for future events and reward follow-up; admin export and purge |

### Why one room

- **Fair comparison.** Every score comes from the same judges watching the same field. With two rooms, each judge sees half the teams; a team in the stronger room is compared against stronger teams and scores lower, and per-judge normalization cannot correct it because no judge saw both halves.
- **Open awards need full coverage.** Nine of the thirteen challenges are decided across every project. Judges can only deliberate those if they all saw every pitch.
- **Teams watch each other.** For first-time hackers, seeing every other project is a large part of what makes them come back.
- **One of everything.** One coordinator, one screen, one Wi-Fi zone, one schedule to keep.
- **Cost: time.** 10 minutes × 8–13 teams is 80–130 minutes. The coordinator absorbs overrun by shortening Q&A; judges score during changeover.

## Event flow and phases

The event moves through six phases, advanced manually by an admin; configured times act as guardrails (the submission lock fires at the close time even if nobody clicks).

1. **Pre-event**: info pages live (overview, challenges, prizes, rules, schedule, judges, resources, announcements). Submissions closed.
2. **Submissions open**: opens at day-1 kickoff. Teams create drafts, edit freely, and mark them submitted.
3. **Submissions closed**: hard lock \~1 hour before day-2 pitches. Only submitted projects enter judging. Gallery becomes visible to participants and judges. Pitch order published.
4. **Judging**: pitches in randomized order; the coordinator advances "now pitching"; judges score each pitch on their phones.
5. **Deliberation**: results become visible to judges; the panel decides awards.
6. **Winners announced**: winner badges appear in the gallery; the Prizes page shows winners.

Pitch slot (default): 5 min demo and pitch + 3 min Q&A + 2 min scoring and changeover = 10 min. `pitch_slot_minutes` is informational (shown to the coordinator as an expected end time); nothing is enforced.

## Roles and access

Four roles; only staff (judge, coordinator, admin) have logins. A staff user can hold several roles (e.g. Umar as admin + judge).

| Role | How they get in | Can do | Cannot do |
| --- | --- | --- | --- |
| Participant | QR code → event access code → signed cookie; own submission via private edit link | Read info pages, create and edit their submission until the close, view gallery after close | See other drafts, scores, contact data |
| Judge | Email + admin-generated password | Browse projects, follow "now pitching", score, write private notes, edit own scores until finalized | See other judges' scores before deliberation, see contact data |
| Coordinator | Same login | Set pitch order, advance "now pitching" | See scores, results, contact data |
| Admin | Same login | Everything: settings, phases, staff, submissions incl. contact data, results, awards, exports | Nothing |

Access code: shown via the QR code in the room; admin can rotate it if it leaks. Rotating does not invalidate existing edit links.

Edit link: unguessable token shown once on the confirmation screen ("save this link"). Lost link → an organizer looks the team up in admin and reissues one.

Staff passwords: generated by admin (≥ 16 characters, random), shown once. Admin can reset (new password shown once) or disable an account. There is no self-service sign-up or password reset by email.

## Public site pages

Modelled on a Devpost hackathon site. All pages sit behind the access code; static content lives as markdown in the repo, not a CMS.

| Page | Content | Source |
| --- | --- | --- |
| Overview | Hero, dates, venue, countdown to the current phase deadline, latest announcement | Markdown + event settings |
| Challenges | Open challenges, then pitched challenges; each with tagline, what wins it, eligibility, example directions | `challenge` table (admin-editable) |
| Prizes | Prize per challenge; winners after announcement | Markdown + awards |
| Rules | Eligibility, team size, deadline, "built during the event", conduct, one-award policy if set to block | Markdown |
| Schedule | Day 1 and day 2 timeline; pitch order once published | Markdown + pitch state |
| Judges | Name, title, photo for staff marked "show publicly" | `staff_user` |
| Resources | Starter links, git help, submission walkthrough, Discord or WhatsApp | Markdown |
| Announcements | Short organizer posts, newest first | `announcement` |
| Gallery | Project cards with filters by pitched challenge and hardware; detail page per project | Submissions (after close) |

Gallery card and detail page show team name, project name, tagline, story, built-with tags, links, video embed, screenshots, pitched challenges, hardware badge and member first names. Contact fields never appear.

## Challenges

Seeded from the agreed challenge list (final wording from the committee review). Admin can edit text; kind and sort criterion are seed-level settings.

| Slug | Name | Kind | Deliberation sort | Notes |
| --- | --- | --- | --- | --- |
| grand | Grand Prize | open | Overall | Award place 1 |
| runnerup | 2nd and 3rd Place | open | Overall | Award places 2 and 3 |
| impact | Most Impactful Hack | open | Practicality | Proxy only; practicality measures usefulness, not reach |
| innovative | Most Innovative Hack | open | Originality |  |
| technical | Most Technically Complex Hack | open | Technicality |  |
| hardware | Best Hardware Hack | open | Technicality | `hardware_only`: nominees are submissions with `has_hardware` |
| insight | Best User Insight | open | Overall | Judges rely on notes; evidence lives in the pitch and story |
| ihsan | Ihsan Award | open | Aesthetics |  |
| venture | Most Likely to Become a Venture | open | Overall | Award exempt from the multi-award limit by default |
| masjid-app | The Masjid App Challenge | pitched | Overall | Nominees: teams that opted in |
| quran | The Quran Learning Challenge | pitched | Overall |  |
| attention | The Attention Challenge | pitched | Overall |  |
| learning-gap | The Learning Gap Challenge | pitched | Overall |  |

A pitched challenge with a single entrant can still be awarded.

## Submission form and rules

One submission per team, in Devpost's format, with draft and submitted states.

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| Team name | Text | Yes | Soft uniqueness warning (case-insensitive) |
| Members (1–4) | Repeating group: name, email, Discord ID | Name + email yes, Discord optional | Contact fields admin-only |
| Project name | Text | Yes |  |
| Tagline | Text, ≤ 120 chars | Yes | One-line pitch |
| Story | Markdown | Yes | Pre-filled headings: Inspiration, What it does, How we built it, Who we talked to, Challenges, Accomplishments, What we learned, What's next |
| Built with | Tags | No | Free-text tags |
| Repo URL | URL | Yes | Public repo expected |
| Live demo URL | URL | No |  |
| Video demo | URL | No | YouTube, Loom, Drive; embedded where possible |
| Screenshots | Image upload | No | Up to 5, ≤ 5 MB each, JPG/PNG/WebP |
| Pitched challenges | Multi-select of pitched challenges only | No, 0–3 | Hard cap of 3. Helper text: "Every project is automatically considered for all open challenges." |
| Hardware component | Checkbox | No | "Our project includes physical hardware" |

"Who we talked to" is added to the story template because Best User Insight is judged on it.

Rules enforced by the app:

- **Draft vs submitted:** only submitted projects are judged. Teams can edit a submitted project until the close.
- **Hard close:** after the close time, writes are rejected server-side and by database triggers on `submission`, `member`, `screenshot` and `submission_challenge`. Admin can grant a per-team grace override.
- **Pitched only:** a trigger rejects `submission_challenge` rows whose challenge is not `pitched`, and more than 3 per submission.
- **Uploads:** see Screenshot handling.
- **Rate limit** on create and upload per access-code session.

### Screenshot handling

Railway Buckets are private and S3-compatible; public buckets are not supported, so every image is served through a signed URL.

1. Browser posts the file (multipart) to a route handler in the app. No direct presigned uploads, because validation must run before the file is stored.
2. Server checks size (≤ 5 MB) and sniffs the type from the bytes (`file-type`); only JPEG, PNG, WebP pass.
3. `sharp` re-encodes to WebP, caps the long edge at 2000 px, and strips EXIF and other metadata.
4. Server writes the object with a random key (`screens/<uuid>.webp`) via `@aws-sdk/client-s3` and inserts the `screenshot` row.
5. Pages that show screenshots generate presigned GET URLs at render time (1-hour expiry).

Judge photos follow the same path with `staff/<uuid>.webp`.

## Judging

Every judge scores every pitch on five criteria from [Devfolio's judging guide](https://guide.devfolio.co/docs/guide/judging), 0–10 each, equal weights by default (weights configurable in admin). Short anchors appear under each criterion on the judge screen.

| Criterion | Question | 0 | 5 | 10 |
| --- | --- | --- | --- | --- |
| Technicality | Is it functional, and does it justify the hackathon time? | Doesn't run / slides only | Core flow works with rough edges | Works end to end, ambitious for the time |
| Originality | How new is the idea or approach? | A clone of something common | Fresh angle on a known idea | Genuinely novel idea or approach |
| Practicality | Is it useful and accessible to real users? | No clear user | Plausible use for a defined user | Clearly useful, could be used tomorrow |
| Aesthetics | Is the UI/UX intuitive? | Hard to use or no UI | Usable, unpolished | Clean, intuitive, considered |
| Wow factor | Does something make it stand out? | Forgettable | One memorable element | Judges talk about it afterwards |

Pitch-time flow:

1. Coordinator sets the randomized pitch order (published at the close).
2. Coordinator taps "now pitching"; every judge's screen switches to that team within a few seconds. Clients poll `GET /api/pitch-state` every 3 seconds (ETag, 304 when unchanged); no websockets.
3. Judge sees the project summary, links, screenshots, pitched challenges and hardware badge, then enters five scores and optional private notes.
4. Scores save as the judge goes; "Finalize" locks that judge's scores for the team. Admin can unlock.
5. Judges keep a manual project picker in case "now pitching" isn't advanced.
6. A conflict-of-interest toggle lets a judge skip a team; that team's results use the remaining judges.

Judges see only their own scores ("My scores" view) until the Deliberation phase.

## Results and awards

The app produces ranked lists; the judges make the final call. This mirrors ETHPrague, where scores are normalized into a ranked list and the panel then decides winners.

Overall computation, per team:

1. Weighted score per judge = sum of (criterion score × weight) / sum of weights, range 0–10.
2. Normalize per judge: z = (score − judge's mean) / judge's population standard deviation across the teams they scored. This removes harsh vs lenient judging.
3. Overall rank score = mean z across judges who scored the team.
4. Shown alongside: raw mean weighted score, mean rank across judges, number of judges who scored, per-criterion means.

Per-criterion computation (for award sorting), per team and criterion: the same normalization on the single criterion, so a judge who is harsh only on design does not sink a team's Ihsan ranking.

```latex
z_{j,t} = \frac{s_{j,t} - \bar{s}_j}{\sigma_j}, \qquad \text{rank}_t = \frac{1}{|J_t|}\sum_{j \in J_t} z_{j,t}
```

```latex
z_{j,t,c} = \frac{s_{j,t,c} - \bar{s}_{j,c}}{\sigma_{j,c}}, \qquad \text{rank}_{t,c} = \frac{1}{|J_t|}\sum_{j \in J_t} z_{j,t,c}
```

Edge cases: σ = 0 for a judge (overall or on one criterion) gives z = 0 for every team from that judge. A team scored by fewer than 3 judges is flagged. Ties sort by raw mean, then by team name.

Deliberation view (judges, Deliberation phase):

- Overall leaderboard.
- One nominee list per challenge:
  - Open challenges: every submitted project (hardware-only: those with `has_hardware`), sorted by the challenge's sort criterion, or overall if none.
  - Pitched challenges: teams that opted in, sorted by overall.
- Judges' private notes per team, attributed.

Awards (admin):

- Award rows per challenge; Grand Prize is place 1, 2nd and 3rd Place are places 2 and 3; every other challenge has one award. Any award can have several winners (shared places).
- Multi-award policy (`event.award_policy`):
  - `allow`: no check.
  - `warn` (default): assigning a team that already holds a non-exempt award shows a warning and proceeds on confirm.
  - `block`: the assignment is rejected.
  - Awards with `exempt_from_limit` (venture by default) are ignored by the check, in both directions.
- Publishing awards moves the event to Winners announced and adds badges in the gallery.

## Admin console

One `/admin` area; every action is checked server-side against the admin role.

| Area | Capabilities |
| --- | --- |
| Event settings | Name, open and close times, current phase, pitch slot minutes, award policy, rotate access code, show QR code for printing |
| Content | Edit challenge text; post, edit and delete announcements |
| Staff | Add or remove judges, coordinators and admins; name, email, title, photo, "show publicly"; generate password (shown once), reset password, disable account |
| Submissions | List with status (draft, submitted, late); view full details incl. contact data; edit, delete, reissue edit link, grant per-team grace override |
| Pitching | Randomize and reorder pitch list; drive "now pitching" (also available to coordinator) |
| Judging progress | Matrix of judges × teams showing scored / finalized / skipped |
| Results | Leaderboard and per-challenge nominee lists; unlock a judge's finalized scores |
| Awards | Assign winners per award, shared allowed; policy warnings; publish |
| Data | CSV export of submissions, contacts, scores and results; purge contact data (removes emails and Discord IDs, keeps projects) |

## Data model

Postgres, single event per deployment (an `event` row keeps the door open for reuse). Auth library tables (`user`, `session`, `account`, `verification`) are owned by Better Auth and not listed; password hashes live in its `account` table.

| Table | Key fields | Notes |
| --- | --- | --- |
| event | name, phase, opens\_at, closes\_at, access\_code\_hash, pitch\_slot\_minutes (default 10), award\_policy (allow / warn / block, default warn) | Single row; times stored as timestamptz, displayed in America/New\_York |
| challenge | slug (unique), name, kind (open / pitched), tagline, wins\_md, eligibility\_md, examples\_md, sort\_criterion\_id (nullable), hardware\_only (bool), sort\_order | Seeded from the table above |
| submission | team\_name, project\_name, tagline, story\_md, built\_with\[\], repo\_url, demo\_url, video\_url, has\_hardware (bool), status (draft / submitted), edit\_token\_hash, grace\_until, submitted\_at, created\_at, updated\_at | Close trigger |
| submission\_challenge | submission\_id, challenge\_id | Close trigger; trigger enforces pitched-only and max 3 |
| member | submission\_id, name, email, discord\_id | Close trigger; contact fields admin-only; 1–4 per submission |
| screenshot | submission\_id, storage\_key, width, height, bytes, sort\_order | Close trigger; max 5 per submission |
| staff\_user | user\_id (unique FK to auth `user`), name, title, photo\_key, show\_publicly, roles\[\] (judge, coordinator, admin), disabled | Only these users can log in |
| criterion | name, question, anchor\_0, anchor\_5, anchor\_10, weight, sort\_order | Seeded with the five Devfolio criteria |
| score | judge\_id, submission\_id, criterion\_id, value (0–10) | Unique (judge, submission, criterion) |
| judge\_review | judge\_id, submission\_id, notes, finalized\_at, skipped\_conflict | Unique (judge, submission) |
| pitch\_state | order (submission\_id\[\]), current\_submission\_id, updated\_at | Single row |
| award | challenge\_id, name, place (nullable), exempt\_from\_limit (bool), published |  |
| award\_winner | award\_id, submission\_id | Several rows = shared award |
| announcement | title, body\_md, published\_at |  |

Close trigger (all four tables): reject INSERT, UPDATE and DELETE when `now() >= event.closes_at` unless the parent submission's `grace_until > now()`. Admin writes go through the same rule; admin bypass is by setting `grace_until`.

## Security and data handling

The realistic threats are a leaked QR code and a bug that exposes scores or contact data, not a determined attacker. Controls are sized to that.

- **Server-side authorization on every staff action.** Hiding a button is never the control; each judge, coordinator and admin endpoint checks role on the server.
- **No public sign-up.** The auth library's sign-up endpoint is disabled; staff exist only when an admin creates them.
- **Contact data isolation.** Public and judge queries select explicit columns that never include email or Discord ID. Only admin queries touch `member` contact fields.
- **Secrets stored hashed.** Access code and edit tokens with argon2id; staff passwords hashed by the auth library. Generated passwords are shown once.
- **Deadline enforced twice.** Application check plus database triggers, so one layer's bug can't reopen submissions.
- **Score privacy.** Judges read only their own `score` and `judge_review` rows until the Deliberation phase.
- **Uploads.** Server-side validation and re-encoding, random storage keys, private bucket, presigned GET URLs with 1-hour expiry.
- **Abuse.** Rate limits on submission create and upload; admin can delete entries and rotate the access code.
- **Sessions.** Secure, HTTP-only cookies; staff sessions expire after 7 days; disabling a staff account revokes their sessions.
- **Retention.** Contact data kept for future events and reward follow-up; admin export and purge available; legal team owns the retention period.
- **Backups.** Daily database backup during the event week, plus a manual export right after submissions close and after awards. Railway does not back up buckets; screenshots are not backed up.

## Tech stack, infrastructure and cost

One TypeScript Next.js app with Postgres and a storage bucket on Railway.

| Layer | Choice | Why |
| --- | --- | --- |
| App | Next.js (current stable, App Router), TypeScript, server actions + route handlers | One deployable for public, judge and admin areas |
| Database | Railway Postgres | Same platform as the app; supports triggers and constraints |
| ORM and migrations | Drizzle + drizzle-kit | Typed schema; triggers live in raw SQL migrations alongside generated ones |
| Auth | Better Auth: email/password + admin plugin; sign-up disabled | Admin creates users and sets passwords; no social providers |
| Participant access | Signed cookie after access code; hashed edit tokens | No participant accounts |
| File storage | Railway Bucket via `@aws-sdk/client-s3` and `@aws-sdk/s3-request-presigner` | Private, S3-compatible, no external provider |
| Image processing | sharp, file-type | Validate, resize, re-encode, strip metadata |
| Styling | Tailwind + shadcn/ui | Fast, consistent, mobile-first judge UI |
| Tests | Vitest (unit, incl. scoring math), Playwright (smoke flows) | See Testing |
| Hosting | Railway: one web service + Postgres + Bucket | No infra beyond this |

Cost (approximate, Oct 2026; confirm on Railway's pricing page):

| Item | Estimate |
| --- | --- |
| Railway Hobby plan (includes $5 of usage) | $5/month minimum |
| Usage above the included $5 (small service + small Postgres) | $0–10/month |
| Railway Bucket ($0.015 per GB-month, < 1 GB) | \~$0 |
| Total for \~1–2 months around the event | \~$10–30 |

## Implementation notes

### Environment variables

| Variable | Source |
| --- | --- |
| `DATABASE_URL` | Railway Postgres variable reference |
| `BETTER_AUTH_SECRET` | Generated, 32+ bytes |
| `BETTER_AUTH_URL` | Public app URL |
| `ACCESS_COOKIE_SECRET` | Generated, signs the participant access cookie |
| `S3_ENDPOINT`, `S3_REGION`, `S3_BUCKET`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY` | Railway Bucket credentials via variable references |
| `BOOTSTRAP_ADMIN_EMAIL` | Used once by the seed script to create the first admin |

### Seed script

Idempotent `pnpm db:seed`: the event row (phase pre-event, times in America/New\_York), the five criteria, the 13 challenges with their awards, and the bootstrap admin (prints a generated password once).

### Deploy on Railway

1. Project with three services: web (GitHub repo), Postgres, Bucket.
2. Wire variables by reference; set `BETTER_AUTH_URL` to the public domain.
3. Pre-deploy command runs migrations (`drizzle-kit migrate`); seed runs manually once.
4. Custom domain or the Railway subdomain; the QR code encodes `https://<domain>/?code=<access code>`.

### Testing

- **Unit (Vitest):** weighted score, overall and per-criterion z, σ = 0, fewer-than-3-judges flag, tie order, award-policy check incl. exemptions. Verify against a hand-computed fixture for 3 teams × 3 judges.
- **Database:** close triggers on all four tables, grace override, pitched-only and max-3 triggers.
- **Smoke (Playwright):** access code → draft → submit with screenshot → locked after close; judge login → score → finalize; coordinator advance reflected on a judge page within 5 seconds.

## Milestones, effort and fallback

About 4 working days for one senior engineer. Today is Oct 14; the event is Oct 24.

| # | Milestone | Effort | Done when | Target |
| --- | --- | --- | --- | --- |
| 0 | Fallback: Google Form for submissions, printed scoring sheet | 0.25 day | Both exist and are tested by one person | Oct 15 |
| 1 | Scaffold, schema, triggers, migrations, seed, Railway deploy (web + Postgres + Bucket) | 0.5 day | Empty app live with DB and bucket | Oct 15 |
| 2 | Access code gate, public pages, challenges page, announcements | 0.5 day | QR → code → pages render | Oct 16 |
| 3 | Submission create/edit, drafts, screenshot pipeline, close enforcement | 1 day | Team can draft, upload, submit, and is locked out at close | Oct 17 |
| 4 | Staff auth (email/password, admin-created), roles, staff management | 0.5 day | Admin adds a judge; judge logs in | Oct 18 |
| 5 | Judge UI: now pitching, scoring, notes, finalize, my scores | 1 day | Six judges score on phones, scores private | Oct 19 |
| 6 | Admin: phases, pitch control, progress, results, per-challenge nominees, awards with policy, exports, purge | 0.5–1 day | Full event can be run from admin | Oct 20 |
| 7 | Dry run and fixes | 0.5 day | Checklist below passes | Oct 21–22 |
| 8 | Freeze; load final challenge text, judges, access code; print QR | 0.25 day | Nothing but content changes after this | Oct 23 |

Dry run checklist (fake data, 2–3 helpers on their own phones):

- [ ] 10 fake teams submit, including one with 5 screenshots, one with hardware, one in no pitched challenges, and one that misses the deadline
- [ ] Screenshots render in gallery and judge views; image files carry no EXIF
- [ ] Access code rotation does not break existing edit links
- [ ] Adding a member or screenshot after the close fails; grace override allows it
- [ ] Two test judges score all teams; neither can see the other's scores
- [ ] Coordinator advances "now pitching"; judge screens follow within a few seconds
- [ ] Results match a hand calculation for 3 teams, overall and on one criterion
- [ ] Hardware nominee list shows only hardware teams; Ihsan list sorts by aesthetics
- [ ] Award policy warns on a second award and ignores the venture award
- [ ] Admin generates and resets a judge password; disabled judge is logged out
- [ ] Gallery and judge views contain no emails or Discord IDs
- [ ] CSV exports open cleanly; purge removes contact data only
- [ ] Venue Wi-Fi or a phone hotspot loads the judge UI acceptably

Event-day fallback: the Google Form for submissions and a printed scoring sheet per judge, used only if the app is down.

## Out of scope and items to verify

Out of scope for v1.1:

- Registration and ticketing (Summit registration page)
- Participant accounts and team management
- Social login and email-based password reset
- Email sending (announcements page plus the event chat)
- Video and zip uploads (links only)
- Audience voting, certificates, sponsor tooling, pitch timer enforcement
- Multi-event support beyond the single `event` row

To verify at implementation:

- [ ] Better Auth admin plugin supports creating users and setting passwords server-side, and sign-up can be disabled, in the version used
- [ ] Railway Bucket credential variable names match the S3 client configuration
- [ ] All judges' emails collected; passwords handed over in person or by a private message
- [ ] Normalization edge cases as listed in Results
- [ ] Venue Wi-Fi capacity for 40 participants + 6 judges on the app at once
- [ ] Legal decision on under-13 participants' contact data
