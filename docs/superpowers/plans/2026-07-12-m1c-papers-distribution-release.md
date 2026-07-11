# Milestone 1C: Public Papers, Distribution, and Manual Release Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Connect the validated public learning app to safe Supabase RPCs, deliver public past-paper PDF viewing and evidence-scoped exam distribution, then prepare—but do not automatically perform—the public Netlify release.

**Architecture:** Typed repository adapters isolate the React UI from Supabase. TanStack Query handles public reads and short-lived document URLs without persistence. Availability comes from published release state; paper/distribution pages render honest empty/coverage states, and security headers constrain network, frames, scripts, and assets.

**Tech Stack:** React 19.2.7, React Router 7.18.1, TanStack Query 5.101.2, Supabase JS 2.110.2, Zod 4.4.3, native accessible SVG/CSS charts plus data tables, Vitest/MSW/Testing Library, Playwright/Axe, Netlify static hosting.

## Global Constraints

- Prerequisites: Milestone 1A and 1B local gates pass; remote mutations still require a fresh owner confirmation.
- Public pages expose only published safe RPC projections; never query private base tables or use storage paths.
- No public upload, edit, delete, owner login, admin link, or owner identity appears in markup, sitemap, metadata, URL, or client configuration.
- Past-paper PDFs are owner-provided only; no official PDF is bundled or seeded.
- A chart uses only published papers with complete question indexes and always displays its exact coverage numerator, denominator, and sessions.
- Empty data is shown honestly; no synthetic frequencies, sample paper cards, inflated counts, or unfinished-library links.
- SL/HL route, data, filters, progress, and caches remain isolated.
- Signed URLs are requested by document ID, last 60 seconds, use `no-store`, and are never written to local/session storage.
- The public hostname must contain no owner name, email, or GitHub username.
- No GitHub Actions, Netlify continuous deployment, scheduled function, or automatic content publication.
- Stage only named files and preserve unrelated dirty worktree state.

---

## File Structure

```text
scripts/generate-database-types.ts
src/shared/api/{database.types,publicContent.supabase}.ts
src/domain/{pastPaper,distribution}.ts
src/features/past-papers/{PastPapersPage,PastPaperFilters,PublicPdfViewer}.tsx
src/features/exam-distribution/{ExamDistributionPage,DistributionFilters,CoverageDisclosure,DistributionBars,CooccurrenceTable}.tsx
src/features/navigation/{featureAvailability,PublicNavigation}.tsx
src/pages/HomePage.tsx
src/shared/components/{EmptyState,ErrorState}.tsx
e2e/{public-papers,exam-distribution,security-accessibility}.spec.ts
netlify.toml
docs/release-checklist.md
```

### Task 1: Generate database types and implement the safe public repository

**Files:**
- Create: `scripts/generate-database-types.ts`
- Create: `src/shared/api/database.types.ts`
- Create: `src/domain/pastPaper.ts`
- Create: `src/domain/distribution.ts`
- Create: `src/shared/api/publicContent.supabase.ts`
- Test: `src/shared/api/publicContent.supabase.test.ts`

**Interfaces:**

```ts
export interface PublicContentRepository {
  getAvailability(courseId: CourseId): Promise<PublicAvailability>;
  listTopics(courseId: CourseId): Promise<TopicSummary[]>;
  getTopic(courseId: CourseId, topicId: string): Promise<TopicDetail | null>;
  listPastPapers(filters: PastPaperFilters): Promise<PastPaperPage>;
  getSignedDocumentUrl(documentId: string): Promise<{ url: string; expiresAt: string }>;
  getExamDistribution(filters: DistributionFilters): Promise<DistributionResult>;
  submitTakedownRequest(input: TakedownRequestInput): Promise<void>;
}
```

`PastPaperFilters` includes course, year range, sessions, timezones, papers, document role, and topic. `DistributionFilters` additionally includes calculator mode/domain/topic. `DistributionResult.coverage` contains published count, indexed count, indexed paper IDs/labels, and covered sessions.

- [ ] **Step 1: Write failing adapter tests**

```ts
it("requests published papers through the safe RPC", async () => {
  await repository.listPastPapers({ courseId: "aa_sl", yearFrom: 2021, yearTo: 2026 });
  expect(rpc).toHaveBeenCalledWith("list_published_past_papers", expect.objectContaining({ p_course: "aa_sl" }));
});

it("never exposes a storage path", async () => {
  const page = await repository.listPastPapers({ courseId: "aa_hl" });
  expect(JSON.stringify(page)).not.toMatch(/bucket|objectPath|rightsBasis|createdBy/i);
});
```

Also test Zod rejects unexpected/missing RPC fields, course mismatch, invalid dates/marks, and malformed distribution totals.

- [ ] **Step 2: Run tests to verify failure**

Run: `npm test -- src/shared/api/publicContent.supabase.test.ts`

Expected: FAIL because types/adapter do not exist.

- [ ] **Step 3: Generate types and implement adapters**

Run `npx supabase gen types typescript --local` and capture stdout through the execution tool without shell redirection. Review the generated text for private-table leakage, then use `apply_patch` to create/update the committed generated type file. `publicContent.supabase.ts` calls only the approved RPC names and Edge Function URLs, parses every response with Zod, maps snake_case to domain types, and returns generic English errors without logging payloads or signed URLs.

- [ ] **Step 4: Run tests and commit**

Run: `npm test -- src/shared/api/publicContent.supabase.test.ts && npm run typecheck`

Expected: PASS.

```bash
git add scripts/generate-database-types.ts src/shared/api/database.types.ts src/domain/pastPaper.ts src/domain/distribution.ts src/shared/api/publicContent.supabase.ts src/shared/api/publicContent.supabase.test.ts
git commit -m "feat: add typed safe public content repository"
```

### Task 2: Drive navigation and home cards from published availability

**Files:**
- Modify: `src/features/navigation/featureAvailability.ts`
- Modify: `src/features/navigation/PublicNavigation.tsx`
- Modify: `src/features/course/CourseSwitch.tsx`
- Modify: `src/pages/HomePage.tsx`
- Modify: `src/app/createRoutes.tsx`
- Create: `src/shared/components/EmptyState.tsx`
- Create: `src/shared/components/ErrorState.tsx`
- Test: `src/features/navigation/PublicNavigation.test.tsx`
- Test: `src/pages/HomePage.test.tsx`

**Interfaces:**
- Availability is fetched per course and cached under `['availability', courseId]`.
- Milestone 1 enables Home, Knowledge Map, Past Papers, Exam Distribution, and Study Paths only when their publication gate passes. Topic Practice, Mock Papers, and Paper 3 remain absent until later content milestones.

- [ ] **Step 1: Write failing availability tests**

Assert unavailable features do not appear in header, mobile nav, home cards, sitemap-equivalent route links, or search; direct access returns the normal English 404. Assert switching courses recomputes availability and cannot retain an HL-only path in SL.

- [ ] **Step 2: Run tests to verify failure**

Run: `npm test -- src/features/navigation/PublicNavigation.test.tsx src/pages/HomePage.test.tsx`

Expected: FAIL until repository availability is wired.

- [ ] **Step 3: Implement repository-driven gates**

Use TanStack Query with a five-minute availability cache and no persistence. Loading shows a neutral shell; failure exposes Retry without owner/backend details. Displayed counts come only from the safe RPC response and published records.

- [ ] **Step 4: Run tests and commit**

Run: `npm test -- src/features/navigation/PublicNavigation.test.tsx src/pages/HomePage.test.tsx`

Expected: PASS.

```bash
git add src/features/navigation src/features/course/CourseSwitch.tsx src/pages/HomePage.tsx src/app/createRoutes.tsx src/shared/components/EmptyState.tsx src/shared/components/ErrorState.tsx
git commit -m "feat: gate public navigation by published content"
```

### Task 3: Implement public Past Papers and the expiring inline viewer

**Files:**
- Create: `src/features/past-papers/PastPaperFilters.tsx`
- Create: `src/features/past-papers/PastPapersPage.tsx`
- Create: `src/features/past-papers/PublicPdfViewer.tsx`
- Modify: `src/app/createRoutes.tsx`
- Test: `src/features/past-papers/PastPapersPage.test.tsx`
- Test: `src/features/past-papers/PublicPdfViewer.test.tsx`

**Interfaces:**
- Route: `/:course/past-papers`.
- Viewer receives opaque `documentId`; it never receives bucket/path.

- [ ] **Step 1: Write failing list/viewer tests**

Test course/year/session/timezone/paper/document-role/topic filters, question-paper/markscheme grouping, honest empty state, retry, full English copy, signed URL request on open only, no URL persistence, expiry/reopen refresh, withdrawn 404, iframe title, sandbox, and fallback “Open PDF in a new tab” link.

- [ ] **Step 2: Run tests to verify failure**

Run: `npm test -- src/features/past-papers`

Expected: FAIL because pages do not exist.

- [ ] **Step 3: Implement list and viewer**

The viewer stores the URL in component state only, schedules expiry using the repository's `expiresAt`, clears it on close/unmount/course switch, and requests a new URL for every reopen. Use an inline sandboxed frame with a descriptive title and accessible external-open fallback. The list never renders rights notes or owner/source-private fields.

- [ ] **Step 4: Run tests and commit**

Run: `npm test -- src/features/past-papers`

Expected: PASS.

```bash
git add src/features/past-papers src/app/createRoutes.tsx
git commit -m "feat: add public past-paper PDF library"
```

### Task 4: Implement evidence-scoped Exam Distribution

**Files:**
- Create: `src/features/exam-distribution/DistributionFilters.tsx`
- Create: `src/features/exam-distribution/CoverageDisclosure.tsx`
- Create: `src/features/exam-distribution/DistributionBars.tsx`
- Create: `src/features/exam-distribution/CooccurrenceTable.tsx`
- Create: `src/features/exam-distribution/ExamDistributionPage.tsx`
- Modify: `src/app/createRoutes.tsx`
- Test: `src/features/exam-distribution/ExamDistributionPage.test.tsx`
- Test: `src/features/exam-distribution/DistributionBars.test.tsx`

**Interfaces:**
- Route: `/:course/exam-distribution`.
- Every quantitative view receives both result data and the same `coverage` object.

- [ ] **Step 1: Write failing distribution tests**

```ts
it("always discloses indexed coverage", async () => {
  renderPage({ publishedPaperCount: 12, indexedPaperCount: 7, coveredSessions: ["May 2025", "November 2025"] });
  expect(await screen.findByText("7 of 12 published papers are indexed for this view.")).toBeVisible();
  expect(screen.getByText(/May 2025.*November 2025/)).toBeVisible();
});
```

Also assert no chart is calculated when indexed count is zero; marks shares sum from primary topics only; co-occurrence uses canonical pairs; all filters reset on course switch; keyboard/table alternatives expose exact values; chart text never says “all IB exams” or “IB-wide” unless coverage is explicitly complete.

- [ ] **Step 2: Run tests to verify failure**

Run: `npm test -- src/features/exam-distribution`

Expected: FAIL because distribution components do not exist.

- [ ] **Step 3: Implement accessible views**

Render question count, mark share, session frequency, paper mix, domain/topic bars, and co-occurrence as semantic headings plus accessible tables. CSS/SVG bars are visual supplements; table values are authoritative. Keep the coverage disclosure visible above every view and repeat it in exported/print styles.

- [ ] **Step 4: Run tests and commit**

Run: `npm test -- src/features/exam-distribution`

Expected: PASS.

```bash
git add src/features/exam-distribution src/app/createRoutes.tsx
git commit -m "feat: add coverage-aware exam distribution"
```

### Task 5: Add copyright/non-affiliation notice and public takedown flow

**Files:**
- Create: `src/pages/RightsNoticePage.tsx`
- Modify: `src/features/takedown/TakedownPage.tsx`
- Modify: `src/layouts/PublicShell.tsx`
- Modify: `src/app/createRoutes.tsx`
- Test: `src/pages/RightsNoticePage.test.tsx`
- Test: `src/features/takedown/TakedownPage.test.tsx`

**Interfaces:**
- Routes: `/rights` and `/takedown`.
- Takedown submission is private, CAPTCHA-protected, generic, and never auto-withdraws.

- [ ] **Step 1: Write failing notice/form tests**

Assert the notice states the site is independent and not endorsed by the International Baccalaureate; uploaded documents remain their rights holders' property; no personal owner identity/email appears. Assert the form requires requester email, relationship, reason, details, CAPTCHA, and an empty honeypot; success gives a reference without revealing workflow details.

- [ ] **Step 2: Run tests to verify failure**

Run: `npm test -- src/pages/RightsNoticePage.test.tsx src/features/takedown/TakedownPage.test.tsx`

Expected: FAIL until notice/API integration exists.

- [ ] **Step 3: Implement and verify**

Use the public repository method; do not send data to analytics, local storage, or query strings. Footer links only to the neutral rights/takedown pages, never to `/owner`.

- [ ] **Step 4: Run tests and commit**

Run: `npm test -- src/pages/RightsNoticePage.test.tsx src/features/takedown/TakedownPage.test.tsx`

Expected: PASS.

```bash
git add src/pages/RightsNoticePage.tsx src/features/takedown/TakedownPage.tsx src/layouts/PublicShell.tsx src/app/createRoutes.tsx
git commit -m "feat: add public rights and takedown notice"
```

### Task 6: Add CSP/security headers and end-to-end public release gates

**Files:**
- Modify: `netlify.toml`
- Create: `e2e/public-papers.spec.ts`
- Create: `e2e/exam-distribution.spec.ts`
- Create: `e2e/security-accessibility.spec.ts`
- Create: `docs/release-checklist.md`

**Interfaces:**
- Produces a locally verified production build and a manual release checklist; it does not deploy.

- [ ] **Step 1: Write failing browser/security assertions**

Test direct deep links/refesh, SL/HL isolation, public pages without owner/upload controls, PDF open/reopen/expiry, withdrawal denial, coverage disclosure under filters, empty states, keyboard/focus, mobile layout, axe, and absence of owner email in HTML/JS/network URLs.

- [ ] **Step 2: Configure restrictive headers**

Set `Content-Security-Policy` with `default-src 'self'`; scripts self plus `https://challenges.cloudflare.com` and no eval; styles self plus the minimum KaTeX-compatible policy; images self/data only when validated; connect/frame the exact configured Supabase project origin plus `https://challenges.cloudflare.com`; `object-src 'none'`; `base-uri 'none'`; `form-action 'self'`; and `frame-ancestors 'none'`. Add `X-Content-Type-Options:nosniff`, strict referrer policy, restrictive permissions policy, and SPA redirect. At the confirmed manual-release checkpoint, insert the exact public Supabase origin into `netlify.toml`; it is not a secret, but no owner identity or service credential is written.

- [ ] **Step 3: Run the entire release gate**

```bash
npx supabase db reset
npx supabase test db
deno test supabase/functions --allow-env --allow-net=127.0.0.1
npm run typecheck
npm run content:validate
npm test
npm run build
npm run e2e -- --project=chromium
```

Expected: all exit `0`; no skipped/only tests; production bundle search contains no owner email, service-role key, Chinese public copy, official paper files, or public upload control.

- [ ] **Step 4: Complete manual QA and record evidence**

Verify desktop/mobile/keyboard/focus/contrast, signed PDF display and fallback, every filter, zero-data behavior, coverage wording, owner route isolation, non-admin denial, rights form, current Netlify hostname privacy, and legacy IndexedDB availability. Record date and result in `docs/release-checklist.md`.

- [ ] **Step 5: Commit the release gate**

```bash
git add netlify.toml e2e/public-papers.spec.ts e2e/exam-distribution.spec.ts e2e/security-accessibility.spec.ts docs/release-checklist.md
git commit -m "test: verify manual public release"
```

### Manual public release checkpoint — requires fresh owner confirmation

Stop after local verification. Present:

1. The exact Supabase migration/function/configuration actions from Milestone 1B.
2. The exact Netlify manual build/deploy command or UI action.
3. The final non-personal hostname and expected environment-variable names, without secret values.
4. The tested commit SHA and release-checklist result.

Only after the owner confirms may the executor mutate remote Supabase or Netlify state. Do not enable continuous deployment or any scheduled/automatic publication.
