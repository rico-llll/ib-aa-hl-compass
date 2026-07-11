# Milestone 1A: Public Learning Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the Chinese single-file prototype with a tested English AA SL/HL public learning shell, verified full knowledge maps, prerequisite-aware study paths, and course-isolated local mastery.

**Architecture:** A React Router application consumes typed repositories rather than Supabase directly. Versioned structured content is validated and compiled before build; a verified official syllabus source lock is a hard gate for real knowledge-map authoring. Public routes are course-namespaced, future libraries are capability-gated, and progress stays browser-local under course/version/topic keys.

**Tech Stack:** React 19.2.7, TypeScript 7.0.2, Vite 8.1.4, React Router 7.18.1, Zod 4.4.3, React Markdown 10.1.0, remark-math 6.0.0, rehype-katex 7.0.1, KaTeX 0.17.0, Vitest 4.1.10, Testing Library, Playwright 1.61.1.

## Global Constraints

- Governing specification: `docs/superpowers/specs/2026-07-12-ib-aa-sl-hl-platform-design.md`.
- All public and owner interface copy is English.
- One site contains `IB AA SL` and `IB AA HL`; course state and progress never collide.
- No public route contains upload, edit, delete, owner-login, or owner-navigation controls.
- Hypothesis Testing and matrices are absent from all authored/indexed content; Bayes essential-method count is zero in knowledge checkpoints.
- No official question text is copied or lightly paraphrased.
- Knowledge-map completeness is statement-level against the verified 2021 guide plus corrections effective through November 2026.
- No source guide PDF or correction PDF is committed to Git.
- No generated/imported content is published automatically.
- Do not enable GitHub Actions, Netlify continuous deployment, scheduled generation, or automatic publication.
- Preserve the dirty working tree: stage only named files; do not restore the deleted `README.md` or delete `.netlify/`, `package-lock.json`, `public-build/`, `scripts/`, or `tsconfig.app.tsbuildinfo` unless a named step explicitly owns that file.

---

## File Structure

```text
content/
  syllabus/aa-2021-2026/{source-lock.json,statements.json,exclusions.json}
  knowledge/aa-sl/{map.json,paths.json}
  knowledge/aa-sl/<domain>/<topic-slug>/{topic.json,lesson.md,foundation-check.json,exam-style-checkpoint.json}
  knowledge/aa-sl/<domain>/<topic-slug>/worked-examples/<example-id>.md
  knowledge/aa-hl/{map.json,paths.json}
  knowledge/aa-hl/<domain>/<topic-slug>/{topic.json,lesson.md,foundation-check.json,exam-style-checkpoint.json}
  knowledge/aa-hl/<domain>/<topic-slug>/worked-examples/<example-id>.md
  generated/{knowledge-index.json,reports/*.json}
private/syllabus/aa-2021-2026/{guide.pdf,corrections/*.pdf}
scripts/content/{hash-source,read-content,validate-syllabus,validate-graph,validate-markdown,validate-tex,validate-all,build-knowledge-index}.ts
src/app/{App,AppServices,createRoutes}.tsx
src/domain/{course,content,progress}.ts
src/content/schemas/{common,syllabus,knowledge,assessment}.ts
src/content/{loadKnowledgeMap,renderMarkdown}.ts(x)
src/features/course/{courseRoute,CourseSwitch,coursePreference}.ts(x)
src/features/navigation/{PublicNavigation,featureAvailability}.ts(x)
src/features/progress/{keys,mastery,storage,legacyProgressMigration}.ts
src/features/recommendations/recommendNext.ts
src/features/knowledge-map/{graph,KnowledgeMapPage,TopicPage}.ts(x)
src/features/study-paths/StudyPathsPage.tsx
src/layouts/PublicShell.tsx
src/pages/{HomePage,NotFoundPage}.tsx
src/shared/components/{AsyncState,MathContent,PageHeader}.tsx
src/styles/{tokens,global,shell,cards,forms}.css
src/test/{setup,fakeServices}.ts(x)
tests/content/*.test.ts
tests/learning/*.test.ts
e2e/{public-navigation,course-isolation,knowledge-map}.spec.ts
```

### Task 1: Pin the toolchain and establish test/build gates

**Files:**
- Modify: `package.json`
- Modify: `package-lock.json`
- Modify: `vite.config.ts`
- Modify: `tsconfig.app.json`
- Modify: `index.html`
- Modify: `netlify.toml`
- Create: `.nvmrc`
- Create: `vitest.config.ts`
- Create: `playwright.config.ts`
- Create: `src/test/setup.ts`
- Create: `src/test/smoke.test.ts`

**Interfaces:**
- Produces commands `npm run typecheck`, `npm test`, `npm run content:validate`, `npm run content:build`, `npm run build`, and `npm run e2e`.
- Production build calls content validation before TypeScript/Vite.

- [ ] **Step 1: Write the failing smoke test**

```ts
// src/test/smoke.test.ts
import { describe, expect, it } from "vitest";

describe("document shell", () => {
  it("uses English metadata", () => {
    expect(document.documentElement.lang).toBe("en");
    expect(document.title).toBe("IB Mathematics AA Study Compass");
  });
});
```

- [ ] **Step 2: Install exact dependencies and create the test configuration**

Run:

```bash
NPM_CONFIG_CACHE=/tmp/npm-cache npm install --save-exact react-router-dom@7.18.1 @tanstack/react-query@5.101.2 @supabase/supabase-js@2.110.2 zod@4.4.3 react-hook-form@7.81.0 @hookform/resolvers@5.4.0 react-markdown@10.1.0 remark-math@6.0.0 rehype-katex@7.0.1 katex@0.17.0 gray-matter@4.0.3
NPM_CONFIG_CACHE=/tmp/npm-cache npm install --save-dev --save-exact vitest@4.1.10 @vitest/coverage-v8@4.1.10 jsdom@29.1.1 @testing-library/react@16.3.2 @testing-library/jest-dom@6.9.1 @testing-library/user-event@14.6.1 @playwright/test@1.61.1 @axe-core/playwright@4.12.1 fake-indexeddb@6.2.5 msw@2.15.0 tsx@4.23.0 unified@11.0.5 remark-parse@11.0.0 remark-gfm@4.0.1 mdast-util-to-string@4.0.0 @types/katex@0.16.8
```

Set scripts exactly:

```json
{
  "typecheck": "tsc -b",
  "test": "vitest run",
  "test:watch": "vitest",
  "test:coverage": "vitest run --coverage",
  "content:hash": "tsx scripts/content/hash-source.ts",
  "content:validate": "tsx scripts/content/validate-all.ts",
  "content:build": "tsx scripts/content/build-knowledge-index.ts",
  "build": "npm run content:validate && npm run content:build && tsc -b && vite build",
  "e2e": "playwright test"
}
```

Configure Vitest for `jsdom`, `src/test/setup.ts`, and cleared mocks; configure Playwright with `npm run dev -- --port 4173`, Chromium, and base URL `http://127.0.0.1:4173`. Set Vite `base: "/"`. Add the Netlify SPA redirect from `/*` to `/index.html` with status `200`. Set `<html lang="en">`, the exact title above, and an English description.

- [ ] **Step 3: Run the smoke test and gates**

Run: `npm test -- src/test/smoke.test.ts && npm run typecheck`

Expected: PASS, with no TypeScript errors.

- [ ] **Step 4: Commit only owned files**

```bash
git add package.json package-lock.json vite.config.ts tsconfig.app.json index.html netlify.toml .nvmrc vitest.config.ts playwright.config.ts src/test/setup.ts src/test/smoke.test.ts
git commit -m "test: establish application quality gates"
```

### Task 2: Define course types, capability gates, routes, and the English public shell

**Files:**
- Create: `src/domain/course.ts`
- Create: `src/app/AppServices.tsx`
- Create: `src/app/createRoutes.tsx`
- Create: `src/app/App.tsx`
- Modify: `src/main.tsx`
- Create: `src/features/course/courseRoute.ts`
- Create: `src/features/course/coursePreference.ts`
- Create: `src/features/course/CourseSwitch.tsx`
- Create: `src/features/navigation/featureAvailability.ts`
- Create: `src/features/navigation/PublicNavigation.tsx`
- Create: `src/layouts/PublicShell.tsx`
- Create: `src/pages/HomePage.tsx`
- Create: `src/pages/NotFoundPage.tsx`
- Test: `src/features/course/courseRoute.test.ts`
- Test: `src/app/createRoutes.test.tsx`
- Test: `src/layouts/PublicShell.test.tsx`

**Interfaces:**
- Produces `CourseId`, `CourseSlug`, `PublicFeature`, `PublicAvailability`, `courseIdFromSlug()`, `courseSlugFromId()`, `switchCoursePath()`, `AppServices`, and `createRoutes(services)`.
- Route factory supports `/sl`, `/hl`, and capability-gated child routes. At this task all content capabilities are false; disabled routes resolve to the English not-found page. Task 16 enables Knowledge Map and Study Paths only after validated content exists.

- [ ] **Step 1: Write failing course and navigation tests**

```ts
import { describe, expect, it } from "vitest";
import { courseIdFromSlug, switchCoursePath } from "./courseRoute";

describe("course routes", () => {
  it("parses only supported slugs", () => {
    expect(courseIdFromSlug("sl")).toBe("aa_sl");
    expect(courseIdFromSlug("hl")).toBe("aa_hl");
    expect(courseIdFromSlug("ai")).toBeNull();
  });

  it("leaves HL Paper 3 when switching to SL", () => {
    expect(switchCoursePath("/hl/paper-3", "sl")).toBe("/sl/knowledge-map");
  });
});
```

`PublicShell.test.tsx` must render every enabled public route and assert that its visible text contains no CJK code point and no button/link named Upload, Edit, Delete, Owner, Sign in, or Admin. It must assert Paper 3 appears only for HL when its capability is true.

- [ ] **Step 2: Run tests to verify missing modules fail**

Run: `npm test -- src/features/course/courseRoute.test.ts src/app/createRoutes.test.tsx src/layouts/PublicShell.test.tsx`

Expected: FAIL because route modules do not exist.

- [ ] **Step 3: Implement the stable public interfaces**

```ts
// src/domain/course.ts
export type CourseId = "aa_sl" | "aa_hl";
export type CourseSlug = "sl" | "hl";
export type PublicFeature =
  | "knowledgeMap" | "topicPractice" | "pastPapers"
  | "examDistribution" | "mockPapers" | "studyPaths" | "paper3";
export type PublicAvailability = Record<PublicFeature, boolean>;

export const courseIdFromSlug = (slug: string): CourseId | null =>
  slug === "sl" ? "aa_sl" : slug === "hl" ? "aa_hl" : null;
export const courseSlugFromId = (course: CourseId): CourseSlug =>
  course === "aa_sl" ? "sl" : "hl";
```

```ts
// src/features/course/courseRoute.ts
import type { CourseSlug } from "../../domain/course";
export function switchCoursePath(pathname: string, next: CourseSlug): string {
  if (pathname === "/hl/paper-3" && next === "sl") return "/sl/knowledge-map";
  const parts = pathname.split("/");
  if (parts[1] === "sl" || parts[1] === "hl") parts[1] = next;
  return parts.join("/") || `/${next}`;
}
```

`AppServices` contains `publicContent`, `ownerAuth`, `ownerContent`, `progress`, `legacyPapers`, and `clock`. `createRoutes(services)` returns React Router route objects so unit tests use `createMemoryRouter`; production uses `createBrowserRouter`. Owner routes remain outside `PublicShell` and are lazy loaded. Start with every content capability disabled and a real English Home page that reports no fabricated counts; later tasks enable only validated/published features.

- [ ] **Step 4: Run tests and commit**

Run: `npm test -- src/features/course/courseRoute.test.ts src/app/createRoutes.test.tsx src/layouts/PublicShell.test.tsx`

Expected: PASS.

```bash
git add src/domain/course.ts src/app src/main.tsx src/features/course src/features/navigation src/layouts/PublicShell.tsx src/pages/HomePage.tsx src/pages/NotFoundPage.tsx
git commit -m "feat: add English course-aware public shell"
```

### Task 3: Implement course/version-isolated progress and recommendations

**Files:**
- Create: `src/domain/progress.ts`
- Create: `src/features/progress/keys.ts`
- Create: `src/features/progress/mastery.ts`
- Create: `src/features/progress/storage.ts`
- Create: `src/features/progress/legacyProgressMigration.ts`
- Create: `src/features/recommendations/recommendNext.ts`
- Test: `tests/learning/mastery.test.ts`
- Test: `tests/learning/recommendation.test.ts`
- Test: `tests/learning/course-isolation.test.ts`

**Interfaces:**
- Produces `ProgressKey`, `TopicProgress`, `TopicState`, `progressKey()`, `topicState()`, `recommendNext()`, and `LocalProgressRepository`.
- Reads legacy `localStorage["ib-aa-progress"]` without deleting it; writes version 2 only to `localStorage["ib-aa-progress:v2"]`.

- [ ] **Step 1: Write failing mastery tests**

```ts
it("requires 80 percent and a clean correct checkpoint", () => {
  expect(deriveMastery({ bestCompletedFoundationPercent: 80, correctExamStyleWithoutPriorReveal: 1 })).toBe(true);
  expect(deriveMastery({ bestCompletedFoundationPercent: 79.99, correctExamStyleWithoutPriorReveal: 1 })).toBe(false);
  expect(deriveMastery({ bestCompletedFoundationPercent: 100, correctExamStyleWithoutPriorReveal: 0 })).toBe(false);
});
```

Also test that incomplete/lower retries cannot reduce the best score; browsing is not an attempt; attempted locked nodes become `learning`; all direct prerequisites must be mastered for `ready`; recommendation uses earliest path stage, then lowest mean of the last five scores, then syllabus order, then ID.

- [ ] **Step 2: Run tests to verify failure**

Run: `npm test -- tests/learning`

Expected: FAIL because progress functions do not exist.

- [ ] **Step 3: Implement progress keys and mastery**

```ts
export type ProgressKey = `${"aa_sl" | "aa_hl"}:${string}:${string}`;
export interface TopicProgress {
  key: ProgressKey;
  firstAttemptAt?: string;
  bestCompletedFoundationPercent?: number;
  recentCompletedScores: number[];
  correctExamStyleWithoutPriorReveal: number;
  legacyStatus?: "locked" | "ready" | "learning" | "mastered";
}
export const progressKey = (course: "aa_sl" | "aa_hl", version: string, topicId: string): ProgressKey =>
  `${course}:${version}:${topicId}`;
export const deriveMastery = (p: Pick<TopicProgress, "bestCompletedFoundationPercent" | "correctExamStyleWithoutPriorReveal">) =>
  (p.bestCompletedFoundationPercent ?? 0) >= 80 && p.correctExamStyleWithoutPriorReveal >= 1;
```

Legacy manual `mastered` is retained only as `legacyStatus`; it does not satisfy the new mastery rule. Course, version, and topic are validated before save.

- [ ] **Step 4: Run tests and commit**

Run: `npm test -- tests/learning`

Expected: PASS.

```bash
git add src/domain/progress.ts src/features/progress src/features/recommendations tests/learning
git commit -m "feat: add isolated mastery and recommendation logic"
```

### Task 4: Define the content contract and safe mathematical renderer

**Files:**
- Create: `src/domain/content.ts`
- Create: `src/content/schemas/common.ts`
- Create: `src/content/schemas/syllabus.ts`
- Create: `src/content/schemas/assessment.ts`
- Create: `src/content/schemas/knowledge.ts`
- Create: `src/content/renderMarkdown.tsx`
- Create: `src/shared/components/MathContent.tsx`
- Create: `scripts/content/validate-markdown.ts`
- Create: `scripts/content/validate-tex.ts`
- Test: `tests/content/schema.test.ts`
- Test: `tests/content/markdown-tex-safety.test.ts`

**Interfaces:**
- Zod schemas are the runtime source of truth; TypeScript types are inferred.
- Topic IDs match `^aa_(sl|hl)\.[1-5]\.[a-z0-9]+(?:-[a-z0-9]+)*$`.
- Markdown accepts no raw HTML or Markdown images; TeX renders with `trust:false`, `strict:"error"`, `maxExpand:1000`, and HTML+MathML output.

- [ ] **Step 1: Write malicious-content tests**

```ts
it.each([
  "<script>alert(1)</script>",
  "[x](javascript:alert(1))",
  "![x](data:image/svg+xml;base64,PHN2Zy8+)",
  "$\\href{https://evil.example}{x}$",
])("rejects unsafe content: %s", (source) => {
  expect(() => validateMarkdownAndTex(source)).toThrow();
});

it("accepts accessible mathematics", () => {
  expect(validateMarkdownAndTex("For $x>0$, $\\ln x$ is defined.")).toEqual({ ok: true });
});
```

- [ ] **Step 2: Run tests to verify failure**

Run: `npm test -- tests/content/schema.test.ts tests/content/markdown-tex-safety.test.ts`

Expected: FAIL because schemas and validator do not exist.

- [ ] **Step 3: Implement exact schemas**

`KnowledgeTopic` must contain: schema version, namespaced ID, course, syllabus version, domain, order, title, summary, statement IDs, mapped objectives, direct prerequisite IDs, lesson filename, worked-example filenames, foundation-check filename, exam-style-checkpoint filename, explicit derivation policy, calculator expectations, assessment patterns, and error/diagnostic/correction triples. Foundation checks contain 5–8 original supported-answer items and fixed pass percent `80`; every node has one original exam-style checkpoint with `libraryRole: "knowledge_checkpoint"` and no Bayes essential method.

`validateMarkdownAndTex()` parses Markdown AST, rejects raw HTML/images/unsafe URLs, validates every math node with KaTeX, rejects trust-required commands and expressions over 8 KiB, and allows only same-origin or HTTPS links. `MathContent` uses `react-markdown`, `remark-math`, and `rehype-katex`; do not install or configure `rehype-raw`.

- [ ] **Step 4: Run tests and commit**

Run: `npm test -- tests/content/schema.test.ts tests/content/markdown-tex-safety.test.ts`

Expected: PASS.

```bash
git add src/domain/content.ts src/content src/shared/components/MathContent.tsx scripts/content/validate-markdown.ts scripts/content/validate-tex.ts tests/content
git commit -m "feat: define safe structured math content"
```

### Task 5: Lock the governing syllabus sources and build coverage/graph validators

**Files:**
- Modify: `.gitignore`
- Create locally, never commit: `private/syllabus/aa-2021-2026/guide.pdf`
- Create locally, never commit: `private/syllabus/aa-2021-2026/corrections/*.pdf`
- Create: `scripts/content/hash-source.ts`
- Create: `scripts/content/read-content.ts`
- Create: `scripts/content/validate-syllabus.ts`
- Create: `scripts/content/validate-graph.ts`
- Create: `scripts/content/validate-all.ts`
- Create: `tests/content/assertDomainRelease.ts`
- Create: `content/syllabus/aa-2021-2026/source-lock.json`
- Create: `content/syllabus/aa-2021-2026/statements.json`
- Create: `content/syllabus/aa-2021-2026/exclusions.json`
- Test: `tests/content/syllabus-coverage.test.ts`
- Test: `tests/content/prerequisite-graph.test.ts`

**Interfaces:**
- The owner supplies the currently effective full guide and applicable corrections before this task can pass.
- `SourceLock.status` must equal `verified`; every document has its real ID, effective dates, and SHA-256.
- `SyllabusStatement` stores source code/page, scope, domain, original short coverage summary, course applicability, and current/superseded status.
- `assertDomainRelease(input: { course: CourseId; domain: DomainId }): DomainReleaseReport` returns mapped/unmapped/orphan/forbidden/graph/asset counts and throws when any required count is non-zero.

- [ ] **Step 1: Add the private-source exclusion and failing source-lock test**

```ts
it("locks real sources through November 2026", () => {
  const lock = loadSourceLock();
  expect(lock.status).toBe("verified");
  expect(lock.validThrough).toEqual({ year: 2026, session: "November" });
  expect(lock.documents.length).toBeGreaterThan(0);
  expect(lock.documents.every((d) => /^[a-f0-9]{64}$/.test(d.sha256))).toBe(true);
});
```

Add `private/syllabus/` to `.gitignore`. Do not create a fabricated verified lock for tests; use isolated fixtures under `tests/fixtures/content`.

- [ ] **Step 2: Obtain the owner-provided source documents and hash them**

Place the files at the private paths above. Run: `npm run content:hash -- private/syllabus/aa-2021-2026`

Expected: a deterministic report of filename and SHA-256; the command refuses symlinks and non-PDF files.

- [ ] **Step 3: Atomize and independently review the syllabus registry**

Create one record for every in-scope SL statement/bullet and AHL extension. Keep exact source code/page references but paraphrase public summaries. Mark corrections as superseding specific records. Record explicit exclusions for Hypothesis Testing and matrices. A second reviewer checks every source reference before setting the lock to `verified`.

- [ ] **Step 4: Implement graph and coverage validation**

The validator rejects duplicate/orphan IDs, unknown/self/cross-course/cross-version prerequisites, cycles with a readable path, undeclared roots, unreachable nodes, redundant direct edges, path-order violations, SL-to-AHL leakage, missing SL core within HL, and uncovered in-scope statements. It derives successors and topological order by `(syllabusOrder, id)`.

- [ ] **Step 5: Run registry/validator tests and commit only non-licensed metadata**

Run: `npm test -- tests/content/syllabus-coverage.test.ts tests/content/prerequisite-graph.test.ts && npm run content:validate -- --phase registry`

Expected: PASS with a verified, internally consistent statement registry and all invalid graph fixtures rejected. Full map coverage is intentionally evaluated domain-by-domain in Tasks 6–15 and globally in Task 16.

```bash
git add .gitignore scripts/content content/syllabus tests/content
git commit -m "feat: lock and validate AA syllabus coverage"
```

## AA SL map workstream

Each domain task uses the frozen statement registry from Task 5. Every atomic node is a coherent 20–60 minute unit with all required topic metadata, an English lesson with exact H2 sections, at least one fully worked example, a 5–8 item foundation check, and one original exam-style checkpoint. Each checkpoint is outside the 580-question library. Each domain receives separate syllabus and mathematical-solution review records before its commit.

### Task 6: SL Number and Algebra

**Files:** `content/knowledge/aa-sl/number-and-algebra/**`, `content/knowledge/aa-sl/map.json`, `content/knowledge/aa-sl/paths.json`, `tests/content/aa-sl-number-algebra.test.ts`

**Interfaces:** Consumes verified SL-core Domain 1 statements and `assertDomainRelease`; produces reviewed `aa_sl.1.*` nodes and prerequisite edges used by later SL domains.

- [ ] Write a failing release assertion for 100% mapped SL-core Domain 1 statements, valid direct prerequisites, complete node assets, zero forbidden content, and zero Bayes essential methods.
- [ ] Run `npm test -- tests/content/aa-sl-number-algebra.test.ts`; expect FAIL for missing content.
- [ ] Group the verified Domain 1 statements into namespaced nodes, author every required file, and map every objective to statement IDs. Cover numerical representations, approximation, sequences/series, finance, exponent/log rules, proof within SL scope, and the SL binomial theorem only when supported by the locked source.
- [ ] Run `npm run content:validate -- --course aa_sl --domain number_algebra`; expect 100% coverage and zero errors/warnings.
- [ ] **Expected:** the domain test passes; the report has zero unmapped statements, orphan nodes, unsafe content, forbidden methods, and graph errors.
- [ ] Commit with `git add content/knowledge/aa-sl/number-and-algebra content/knowledge/aa-sl/map.json content/knowledge/aa-sl/paths.json tests/content/aa-sl-number-algebra.test.ts && git commit -m "content: add reviewed SL number and algebra map"`.

### Task 7: SL Functions

**Files:** `content/knowledge/aa-sl/functions/**`, `content/knowledge/aa-sl/map.json`, `content/knowledge/aa-sl/paths.json`, `tests/content/aa-sl-functions.test.ts`

**Interfaces:** Consumes verified SL-core Domain 2 statements plus Task 6 algebra nodes; produces reviewed `aa_sl.2.*` nodes for trigonometry/calculus prerequisites.

- [ ] Write a failing assertion for all locked SL-core Domain 2 statements, graph reachability from Domain 1 prerequisites, and complete instructional assets.
- [ ] Run `npm test -- tests/content/aa-sl-functions.test.ts`; expect FAIL.
- [ ] Author concepts/domain/range, function notation, graphs/features, transformations, composite/inverse relationships, supported polynomial/reciprocal/exponential/logarithmic functions, equations, and modelling exactly to the locked source.
- [ ] Run `npm run content:validate -- --course aa_sl --domain functions`; expect full coverage and zero errors/warnings.
- [ ] **Expected:** the domain test passes; every Domain 2 statement/objective is mapped and all cross-domain prerequisites resolve.
- [ ] Commit with `git add content/knowledge/aa-sl/functions content/knowledge/aa-sl/map.json content/knowledge/aa-sl/paths.json tests/content/aa-sl-functions.test.ts && git commit -m "content: add reviewed SL functions map"`.

### Task 8: SL Geometry and Trigonometry

**Files:** `content/knowledge/aa-sl/geometry-and-trigonometry/**`, `content/knowledge/aa-sl/map.json`, `content/knowledge/aa-sl/paths.json`, `tests/content/aa-sl-geometry-trigonometry.test.ts`

**Interfaces:** Consumes verified SL-core Domain 3 statements and locked algebra/function prerequisites; produces reviewed `aa_sl.3.*` nodes.

- [ ] Write a failing assertion for all locked SL-core Domain 3 statements, valid function/circular-measure prerequisites, and complete instructional assets.
- [ ] Run `npm test -- tests/content/aa-sl-geometry-trigonometry.test.ts`; expect FAIL.
- [ ] Author measurement, coordinate geometry, circular measure, sine/cosine rules, unit-circle relations, supported identities/equations/models, and SL vectors/lines only to the verified source.
- [ ] Run `npm run content:validate -- --course aa_sl --domain geometry_trigonometry`; expect full coverage and zero errors/warnings.
- [ ] **Expected:** the domain test passes with complete Domain 3 coverage and no unsupported vector/trigonometry technique.
- [ ] Commit with `git add content/knowledge/aa-sl/geometry-and-trigonometry content/knowledge/aa-sl/map.json content/knowledge/aa-sl/paths.json tests/content/aa-sl-geometry-trigonometry.test.ts && git commit -m "content: add reviewed SL geometry and trigonometry map"`.

### Task 9: SL Statistics and Probability

**Files:** `content/knowledge/aa-sl/statistics-and-probability/**`, `content/knowledge/aa-sl/map.json`, `content/knowledge/aa-sl/paths.json`, `tests/content/aa-sl-statistics-probability.test.ts`

**Interfaces:** Consumes verified SL-core Domain 4 statements and supporting algebra/function nodes; produces reviewed `aa_sl.4.*` nodes with forbidden-method counts fixed at zero.

- [ ] Write a failing assertion for all locked SL-core Domain 4 statements, zero Hypothesis Testing, and zero Bayes-essential checkpoints.
- [ ] Run `npm test -- tests/content/aa-sl-statistics-probability.test.ts`; expect FAIL.
- [ ] Author sampling/data, descriptive statistics, correlation/regression, probability/conditional probability, discrete random variables/expectation, and supported binomial/normal distributions. Do not name or require Bayes' theorem.
- [ ] Run `npm run content:validate -- --course aa_sl --domain statistics_probability`; expect full coverage and zero errors/warnings.
- [ ] **Expected:** the domain test passes with complete Domain 4 coverage, zero Hypothesis Testing, and zero Bayes-essential checkpoints.
- [ ] Commit with `git add content/knowledge/aa-sl/statistics-and-probability content/knowledge/aa-sl/map.json content/knowledge/aa-sl/paths.json tests/content/aa-sl-statistics-probability.test.ts && git commit -m "content: add reviewed SL statistics and probability map"`.

### Task 10: SL Calculus

**Files:** `content/knowledge/aa-sl/calculus/**`, `content/knowledge/aa-sl/map.json`, `content/knowledge/aa-sl/paths.json`, `tests/content/aa-sl-calculus.test.ts`

**Interfaces:** Consumes verified SL-core Domain 5 statements and function prerequisites; produces reviewed `aa_sl.5.*` nodes and a complete SL graph/path release candidate.

- [ ] Write a failing assertion for all locked SL-core Domain 5 statements, valid function prerequisites, and no AHL technique.
- [ ] Run `npm test -- tests/content/aa-sl-calculus.test.ts`; expect FAIL.
- [ ] Author derivative concepts/rules, tangents/normals, applications/optimization, antiderivatives/definite integrals, area, kinematics, and only the additional methods present in the locked SL source.
- [ ] Run `npm run content:validate -- --course aa_sl --domain calculus`; expect full coverage and zero errors/warnings.
- [ ] **Expected:** the domain test passes with complete Domain 5 coverage and zero AHL leakage.
- [ ] Commit with `git add content/knowledge/aa-sl/calculus content/knowledge/aa-sl/map.json content/knowledge/aa-sl/paths.json tests/content/aa-sl-calculus.test.ts && git commit -m "content: add reviewed SL calculus map"`.

## AA HL map workstream

Use separate `aa_hl.*` topic IDs even for shared mathematics. HL must map every `sl_core` statement plus every `ahl_extension` statement, while remaining inside the locked source. Follow the same complete asset and two-reviewer rules as SL.

### Task 11: HL Number and Algebra

**Files:** `content/knowledge/aa-hl/number-and-algebra/**`, `content/knowledge/aa-hl/{map,paths}.json`, `tests/content/aa-hl-number-algebra.test.ts`

**Interfaces:** Consumes all core/AHL Domain 1 statements; produces course-namespaced reviewed HL algebra/complex prerequisites.

- [ ] Write the failing complete Domain 1 coverage/asset/graph assertion; run `npm test -- tests/content/aa-hl-number-algebra.test.ts` and expect FAIL.
- [ ] Author all verified core and AHL Domain 1 material, including counting, proof, and complex-number extensions only where the source supports them; exclude matrices and avoid promoting complex locus as a frequent pattern.
- [ ] Run `npm run content:validate -- --course aa_hl --domain number_algebra`; expect full coverage and zero errors/warnings.
- [ ] **Expected:** the domain test passes with complete core/AHL Domain 1 coverage, no matrices, and validated complex-number prerequisites.
- [ ] Commit with `git add content/knowledge/aa-hl/number-and-algebra content/knowledge/aa-hl/map.json content/knowledge/aa-hl/paths.json tests/content/aa-hl-number-algebra.test.ts && git commit -m "content: add reviewed HL number and algebra map"`.

### Task 12: HL Functions

**Files:** `content/knowledge/aa-hl/functions/**`, `content/knowledge/aa-hl/map.json`, `content/knowledge/aa-hl/paths.json`, `tests/content/aa-hl-functions.test.ts`

**Interfaces:** Consumes all core/AHL Domain 2 statements and Task 11 prerequisites; produces reviewed HL function nodes.

- [ ] Write the failing complete Domain 2 assertion; run `npm test -- tests/content/aa-hl-functions.test.ts` and expect FAIL.
- [ ] Author all verified core/AHL function material, with deeper polynomial and rational structures, parameter behavior, transformations, composition/inversion, and only source-supported techniques.
- [ ] Run `npm run content:validate -- --course aa_hl --domain functions`; expect full coverage and zero errors/warnings.
- [ ] **Expected:** the domain test passes with complete core/AHL Domain 2 coverage and valid prerequisite edges.
- [ ] Commit with `git add content/knowledge/aa-hl/functions content/knowledge/aa-hl/map.json content/knowledge/aa-hl/paths.json tests/content/aa-hl-functions.test.ts && git commit -m "content: add reviewed HL functions map"`.

### Task 13: HL Geometry and Trigonometry

**Files:** `content/knowledge/aa-hl/geometry-and-trigonometry/**`, `content/knowledge/aa-hl/map.json`, `content/knowledge/aa-hl/paths.json`, `tests/content/aa-hl-geometry-trigonometry.test.ts`

**Interfaces:** Consumes all core/AHL Domain 3 statements and locked algebra/function nodes; produces reviewed HL geometry/trigonometry nodes.

- [ ] Write the failing complete Domain 3 assertion; run `npm test -- tests/content/aa-hl-geometry-trigonometry.test.ts` and expect FAIL.
- [ ] Author all verified core/AHL trigonometry, models, vectors, lines, and planes. Make complex-number dependencies point to circular-measure/trigonometry nodes where pedagogically required.
- [ ] Run `npm run content:validate -- --course aa_hl --domain geometry_trigonometry`; expect full coverage and zero errors/warnings.
- [ ] **Expected:** the domain test passes with complete core/AHL Domain 3 coverage and valid cross-domain dependencies.
- [ ] Commit with `git add content/knowledge/aa-hl/geometry-and-trigonometry content/knowledge/aa-hl/map.json content/knowledge/aa-hl/paths.json tests/content/aa-hl-geometry-trigonometry.test.ts && git commit -m "content: add reviewed HL geometry and trigonometry map"`.

### Task 14: HL Statistics and Probability

**Files:** `content/knowledge/aa-hl/statistics-and-probability/**`, `content/knowledge/aa-hl/map.json`, `content/knowledge/aa-hl/paths.json`, `tests/content/aa-hl-statistics-probability.test.ts`

**Interfaces:** Consumes all core/AHL Domain 4 statements and supporting nodes; produces reviewed HL statistics/probability nodes with knowledge-checkpoint Bayes count zero.

- [ ] Write the failing complete Domain 4 assertion, including zero Hypothesis Testing and zero Bayes-essential knowledge checkpoints; run `npm test -- tests/content/aa-hl-statistics-probability.test.ts` and expect FAIL.
- [ ] Author all verified core/AHL probability and distribution content, including continuous random variables, PDF/CDF integration, expectation/variance, and parameter interpretation. Explain reverse conditional probability accessibly but reserve the five essential-method Bayes questions for the later HL Topic Practice milestone.
- [ ] Run `npm run content:validate -- --course aa_hl --domain statistics_probability`; expect full coverage and zero errors/warnings.
- [ ] **Expected:** the domain test passes with complete core/AHL Domain 4 coverage, zero Hypothesis Testing, and zero knowledge-checkpoint Bayes essentials.
- [ ] Commit with `git add content/knowledge/aa-hl/statistics-and-probability content/knowledge/aa-hl/map.json content/knowledge/aa-hl/paths.json tests/content/aa-hl-statistics-probability.test.ts && git commit -m "content: add reviewed HL statistics and probability map"`.

### Task 15: HL Calculus

**Files:** `content/knowledge/aa-hl/calculus/**`, `content/knowledge/aa-hl/map.json`, `content/knowledge/aa-hl/paths.json`, `tests/content/aa-hl-calculus.test.ts`

**Interfaces:** Consumes all core/AHL Domain 5 statements and function/trigonometry prerequisites; produces a complete reviewed HL graph/path release candidate.

- [ ] Write the failing complete Domain 5 assertion; run `npm test -- tests/content/aa-hl-calculus.test.ts` and expect FAIL.
- [ ] Author all verified core/AHL limits/continuity, differentiation/applications, integration/area/volume/techniques, differential equations, and series/Maclaurin material; no method may exceed the locked guide.
- [ ] Run `npm run content:validate -- --course aa_hl --domain calculus`; expect full coverage and zero errors/warnings.
- [ ] **Expected:** the domain test passes with complete core/AHL Domain 5 coverage and no out-of-syllabus technique.
- [ ] Commit with `git add content/knowledge/aa-hl/calculus content/knowledge/aa-hl/map.json content/knowledge/aa-hl/paths.json tests/content/aa-hl-calculus.test.ts && git commit -m "content: add reviewed HL calculus map"`.

### Task 16: Compile indexes and implement Knowledge Map, Topic Page, Home, and Study Paths

**Files:**
- Create: `scripts/content/build-knowledge-index.ts`
- Create: `src/content/loadKnowledgeMap.ts`
- Create: `src/features/knowledge-map/graph.ts`
- Create: `src/features/knowledge-map/KnowledgeMapPage.tsx`
- Create: `src/features/knowledge-map/TopicPage.tsx`
- Create: `src/features/study-paths/StudyPathsPage.tsx`
- Modify: `src/pages/HomePage.tsx`
- Create: `src/shared/components/AsyncState.tsx`
- Modify: `src/app/createRoutes.tsx`
- Modify: `src/style.css`
- Create: `src/styles/tokens.css`
- Create: `src/styles/global.css`
- Create: `src/styles/shell.css`
- Create: `src/styles/cards.css`
- Create: `src/styles/forms.css`
- Test: `src/features/knowledge-map/KnowledgeMapPage.test.tsx`
- Test: `src/features/study-paths/StudyPathsPage.test.tsx`
- Test: `src/pages/HomePage.test.tsx`

**Interfaces:**
- `build-knowledge-index` writes deterministic course indexes and coverage/graph reports; source files remain authoritative.
- Public loaders return only validated/reviewed content for the selected course.

- [ ] **Step 1: Write failing UI tests**

Assert five domains, search, direct prerequisite/successor display, all teaching sections, worked examples, foundation checks, exam checkpoints, loading/error/retry, course isolation, locked-node browsing, and recommendation order. Assert disabled future libraries are absent from navigation/home.

- [ ] **Step 2: Run tests to verify failure**

Run: `npm test -- src/features/knowledge-map src/features/study-paths src/pages/HomePage.test.tsx`

Expected: FAIL because pages/loaders do not exist.

- [ ] **Step 3: Build deterministic content and pages**

Run `npm run content:build` and commit generated index/reports only after their source checksums match. Implement responsive graph/list exploration with keyboard focus, accessible HTML+MathML, browser-local checks, and English empty/error states. Students may open locked nodes, but the recommendation engine never recommends them.

- [ ] **Step 4: Run tests and commit**

Run: `npm run content:validate && npm run content:build && npm test -- src/features/knowledge-map src/features/study-paths src/pages/HomePage.test.tsx`

Expected: PASS, with 100% course coverage reports.

```bash
git add scripts/content/build-knowledge-index.ts content/generated src/content/loadKnowledgeMap.ts src/features/knowledge-map src/features/study-paths src/pages/HomePage.tsx src/shared/components/AsyncState.tsx src/app/createRoutes.tsx src/style.css src/styles
git commit -m "feat: deliver complete SL and HL learning maps"
```

### Task 17: Run public learning release gates

**Files:**
- Create: `e2e/public-navigation.spec.ts`
- Create: `e2e/course-isolation.spec.ts`
- Create: `e2e/knowledge-map.spec.ts`
- Create: `docs/development.md`

**Interfaces:**
- Produces a testable local Milestone 1A build; it does not deploy.

- [ ] **Step 1: Add end-to-end release assertions**

Test direct deep links, English-only public copy, no public owner/upload controls, SL/HL switching, separate progress, keyboard traversal/focus, locked-node browsing without recommendation, complete topic sections, safe math, mobile navigation, and axe violations.

- [ ] **Step 2: Run the complete gate**

```bash
npm run typecheck
npm run content:validate
npm test
npm run build
npm run e2e -- --project=chromium
```

Expected: all commands exit `0`; no skipped/only tests; coverage and graph reports have zero gaps, warnings, AHL leakage, forbidden content, or unsafe markup.

- [ ] **Step 3: Manually inspect the production build**

Check desktop and mobile layouts, keyboard order, visible focus, contrast, formula overflow, long titles, five domain filters, direct links, browser refresh, and separate SL/HL progress. Record results in `docs/development.md`.

- [ ] **Step 4: Commit the release gate**

```bash
git add e2e docs/development.md
git commit -m "test: verify public learning foundation"
```

Do not deploy or enable continuous deployment. Continue with Milestone 1B only after this local gate passes.
