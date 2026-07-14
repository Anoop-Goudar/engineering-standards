# AGENTS.md — project guidelines

<!--
HOW TO USE THIS TEMPLATE

1. Copy this file to the root of the new repo as `AGENTS.md`.
2. Create a one-line `CLAUDE.md` next to it containing exactly: @AGENTS.md
   (Claude Code reads CLAUDE.md; other agents read AGENTS.md. One source of truth.)
3. Delete every Stack Module in Part 2 except the one you're building.
4. Fill in Part 5 (Project Brief). It is the only section that should differ
   between projects.
5. Delete this comment block.

WHY IT'S SHORT: this file is injected into the model's context on every single
turn. Length is not free — a 400-line rulebook gets diluted and skimmed. Detail
belongs in docs/ (Part 4), which is read on demand. Keep this file under ~200
lines. If a rule can be enforced by a linter, a type, or CI, enforce it there
and delete the prose.
-->

## Part 1 — Universal (never delete; true for every project)

### Working agreements for AI agents

- **Read before you write.** Check `docs/ARCHITECTURE.md` and the existing
  `components/` + `lib/` (or equivalent) before adding anything. Match the
  conventions already in the codebase — naming, file layout, error handling,
  test style — unless there's a stated reason to deviate.
- **Reuse > compose > create.** Don't add a dependency that an existing one
  already covers. Don't extract a utility until it has a second caller. Solve
  the problem in front of you, not the hypothetical one.
- **Prefer incremental change to rewrite.** Before refactoring, confirm the
  existing code doesn't already satisfy the requirement. Rewrites happen only
  when explicitly asked for.
- **Ambiguity has two rules.** Small UX details: pick a sensible modern default
  and keep moving. Architecture, security, data model, money, or business logic:
  **stop and ask.** Guessing there is how you get a rewrite.
- **Finish vertical slices.** One feature complete — UI + API + loading /
  validation / error states, working end to end — beats four features at 70%.
- **Leave no debris.** Do not create status reports, summary markdown, or
  `verify-*.js` scripts at the repo root. Findings go in the PR description or
  the conversation. Scratch work goes in a scratch dir, not the repo. If work is
  deferred, it goes in `docs/BACKLOG.md` — nowhere else.
- **Comment the why, never the what.** A comment that explains what the next
  line does, or narrates the change you just made, is noise the moment the PR
  merges.

### Code quality

- Strict typing (TS `strict`, Python type hints + `py.typed`). Lint + format on
  save; CI fails on lint errors.
- No dead code, no commented-out code, no magic numbers, no `data2` / `temp` /
  `handleClick2`. Names say what the thing *is*: `UserProfileCard`,
  `useProjects`, `createProject`, `extract_patient_name`.
- Split a file when it clearly does more than one job — layout *and* fetching
  *and* business rules in one file is the signal. Line count is not.

### Security (non-negotiable — assume personal/health data)

- Secrets never reach the client and never enter git. `.env.example` lists every
  key with a placeholder; `.env` is gitignored, always.
- **Authorize on the server, on every protected endpoint.** A hidden button is
  not authorization. Ownership checks (does *this* user own *this* row?) are as
  important as authentication.
- Validate on the client for UX and on the server for truth. Never trust client
  input. Ever.
- Sanitize input, escape output (XSS), CSRF-protect state-changing requests,
  rate-limit anything an authenticated user can call in a loop — especially
  **LLM endpoints**, where the exposure is cost, not just abuse.
- Never log sensitive personal data (health details, documents, tokens, payment
  data) in plaintext. Log the route, the user id, the action — not the payload.

### Errors and observability

- Distinguish network failure / server error / validation error / unauthorized /
  timeout, and say something useful for each. "Something went wrong" is a last
  resort, not a default.
- Wrap major page sections in error boundaries so one broken widget doesn't
  blank the page.
- Log unexpected exceptions and unhandled rejections with enough context to
  debug.

### Standard external services (these are settled — don't re-litigate)

| Concern | Choice |
|---|---|
| LLM | **OpenAI.** Structured Outputs for anything parsed. Never swap providers without an explicit decision. |
| Database | **Postgres** (Neon or Supabase). |
| Analytics | **PostHog.** |
| Object storage | **S3** (presigned URLs; never proxy bytes through the API). |
| Hosting | **Vercel** for the web frontend, **Railway** for APIs, workers, and Python services. |

### Testing

Every feature ships with: happy path, validation failure, error path, and the
obvious edge case. Unit-test logic-heavy functions (matchers, extractors,
parsers, money, dates). Don't unit-test trivial UI wrappers. E2E covers the
critical user journeys only.

### Repository conventions

- `README.md` (setup + run, accurate), `.env.example`, `docs/ARCHITECTURE.md`
  (how it fits together), `docs/BACKLOG.md` (deferred work, with enough context
  to pick up cold).
- Branches: work on `dev`, merge to `main`. **`main` is what deploys to prod.**
- Commits: one-line imperative summary, ≤72 chars, explaining the change in
  product terms. Pick conventional prefixes (`feat:` / `fix:` / `docs:`) or
  plain sentences — but be consistent within a repo, and follow whatever the
  existing `git log` already does.

---

## Part 2 — Stack modules (keep ONE, delete the rest)

<!-- MODULE: Next.js web app -->
### Next.js (App Router)

- TypeScript, Tailwind, shadcn/ui. Compose shadcn primitives before hand-rolling.
- **Server Components by default.** A Client Component needs a reason:
  interactivity, state, or a browser API.
- **Components never fetch and never hold business logic.** They call a service
  layer. On the server that's a direct call; on the client it's via a hook that
  wraps the service. Pick *one* client-fetching approach for the repo and stick
  to it — don't mix.
- Forms: React Hook Form + Zod. Never hand-roll a form of meaningful size with
  `useState`.
- Performance: `next/image`, lazy-load below-the-fold and heavy libs, minimize
  client JS. Memoize only where it measurably helps.

<!-- MODULE: Backend API -->
### Backend API service

- Pick the framework once and say so here (Hono / NestJS / FastAPI). Don't mix
  paradigms inside one service.
- **All business logic lives in the API**, not in frontend server actions. The
  frontend is a client of the API, even when they share a repo.
- Every route: authenticated, authorized, input-validated (Zod / class-validator
  / Pydantic), typed response. Auth guards are applied at the router level and
  audited — verify no subtree accidentally inherits or escapes them.
- Long-running work (extraction, OCR, LLM calls over documents) goes on a **job
  queue with a worker**, not in the request path.
- Passwords, only if you actually have them: argon2/bcrypt, never logged. OAuth
  is preferred — then this bullet is dead and should be deleted.

<!-- MODULE: Python service / library -->
### Python service or library

- `src/` layout, `pyproject.toml`, type hints, `py.typed`. Core deps minimal;
  everything else behind optional extras (`[storage]`, `[demo]`, `[dev]`).
- Plugin/strategy seams over `if`-chains when a pipeline has per-domain
  behavior.
- pytest with real fixtures (sample documents + ground truth), not mocks that
  assert the mock.
- If it's consumed over HTTP, the service contract is versioned and documented
  in `docs/ARCHITECTURE.md`.

<!-- MODULE: React Native / Expo mobile -->
### React Native (Expo)

- Platform components, not web idioms. Ignore all web-only UX rules below.
- Navigation via React Navigation; deep links and notification taps must land on
  the right screen from a cold start.
- Offline and slow-network are the *default* case: every screen has a loading,
  empty, error, and offline state.
- Secrets in `expo-secure-store`, never AsyncStorage.
- OTA vs. binary: know which changes need a store submission before you promise
  a fix.

---

## Part 3 — UX standards (web; skip for mobile module)

**Interaction basics — the stuff that keeps getting missed:**

- Every clickable element: `cursor: pointer` + a visible hover state.
- Every button: distinct hover / focus / loading / disabled states. **Disable
  during an in-flight action** — a spinner alone doesn't stop a double submit.
- Every async operation shows feedback: skeletons for pages/tables/cards, inline
  spinners for buttons. Never a blank screen.
- Every list or table has an **empty state** (one line of explanation + the
  primary action) and an **error state** (human-readable, never a stack trace).
- Successful actions get a toast. Dialogs representing a completed task close
  themselves.

**Use modern components, not browser defaults:** no native `<select>` (use a
Select / Command / Popover, searchable when the list is long); no `alert()` /
`confirm()` / `prompt()` (use AlertDialog / Dialog / Drawer on mobile).

**Tables:** search, sort, pagination, sticky header, skeleton, empty state,
responsive. **Search:** debounced, Escape to close, arrow-key navigation, with
loading / no-results states. Default to cards or a list unless the data is
genuinely tabular or comparative.

**Navigation:** one persistent pattern (top nav *or* sidebar — see Part 5).
Breadcrumbs on nested pages, current section highlighted, mobile collapses to a
drawer or bottom bar. Command palette (⌘K) only when there's enough surface area
to justify it.

**Accessibility (WCAG AA floor):** full keyboard navigation, visible focus
rings, ARIA labels where semantics aren't obvious, sufficient contrast, and
**never convey meaning with color alone.**

---

## Part 4 — Design system defaults

**These are defaults, not law. Part 5 overrides any of them.** If the Project
Brief says "no shadows, no accent color," that wins — it is not a violation.

- **Spacing (px):** 4, 8, 12, 16, 20, 24, 32, 40, 48, 64.
- **Radius:** pick one scale and never mix radii across similar components.
- **Shadow:** none / sm / md. Nothing decorative.
- **Motion:** 150–250ms, ease-out. Fade, slide, scale. Purposeful, never flashy.
- **Containers:** 768px reading/forms · 1280px default · 1440px dense dashboards.
- **Type:** Inter. H1 36 / H2 30 / H3 24 / H4 20 / Body 16 / Small 14.
- **Icons:** Lucide, at 16/18/20/24.
- **Color:** neutral base + one accent + success/warning/destructive. No rainbow.
  Dark mode via semantic tokens, never inverted hex, never pure black.

If the project has real visual ambition, put the full language in
`docs/DESIGN.md` and make Part 5 point at it. That file — not this one — becomes
the source of truth for the look.

---

## Part 5 — Project brief (the only per-project section)

- **Name / one-line purpose:**
- **Who uses it:**
- **Visual personality (1–3 words):**
- **Closest reference for feel (vibe, not a template):**
- **Accent color:**
- **Display font (if not Inter):**
- **Density:** spacious ←→ data-dense
- **Navigation:** top nav / sidebar / minimal
- **Dark mode:** required / nice-to-have / not needed
- **Explicitly off-limits:** (e.g. "no gradients," "no badges," "clinical, not playful")
- **Full design language:** `docs/DESIGN.md` — read it before any UI work. ← delete if none

If this section is blank, default to clean, spacious, neutral —
Linear / Stripe / Notion adjacent. That's a fallback, not a mandate.

---

## Pre-flight — self-check before calling a feature done

**Required:** loading state · empty state · error state · client *and* server
validation · server-side authorization · responsive (mobile/tablet/desktop) ·
keyboard accessible · error messages a human can act on · no console errors ·
no lint warnings · no debris files left in the repo

**Nice-to-have (never let these delay the required list):** dark mode ·
analytics event · command palette · tests for trivial UI wrappers

---

## Precedence

Explicit instructions in the conversation > Part 5 (project brief) + `docs/` >
Parts 1–4. A conversational override applies to **that session only** — don't
silently edit this file to match it. If a rule here is consistently wrong for
this project, say so and change it deliberately.
