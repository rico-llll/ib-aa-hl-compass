# Milestone 1B: Owner Backend and Private Library Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the local-testable Supabase backend and unlinked English owner area for OTP-protected administration, private PDF storage, reviewed publication, question indexing, owner mock uploads, legacy browser-file import, audit, and takedown intake.

**Architecture:** All application tables live in a non-exposed `private` schema. Browser roles cannot read base tables; public and owner operations use narrow, fixed-`search_path` RPCs, while private storage objects are addressed only by opaque document IDs. Upload is a two-phase signed-ticket/finalize flow, publication is rights-gated and audited, and external Supabase configuration/deployment remains manual.

**Tech Stack:** Supabase CLI 2.109.1, PostgreSQL/RLS/pgTAP, Supabase Auth email OTP, Supabase Storage, Supabase Edge Functions (Deno), `@supabase/supabase-js` 2.110.2, React Hook Form 7.81.0, Zod 4.4.3, Vitest/MSW/Testing Library.

## Global Constraints

- Prerequisite: Milestone 1A tests and build pass.
- `/owner` is unlinked and outside the public layout; hiding it is never authorization.
- OTP is exactly six digits; account creation is disabled; only a pre-created admin UUID is authorized.
- Owner email is never bundled, placed in URLs, persisted in browser storage, or sent to analytics.
- The browser receives only `VITE_SUPABASE_URL`, `VITE_SUPABASE_PUBLISHABLE_KEY`, and the public CAPTCHA site key; no service-role secret uses a `VITE_` name.
- Anonymous and authenticated browser roles have no base-table reads or writes.
- PDF bucket is private; never call `getPublicUrl`.
- Maximum file size is 52,428,800 bytes; PDF extension, MIME, `%PDF-` signature, expected size, and SHA-256 are all checked.
- Past-paper records may publish with one or both ready document roles; if both exist they are paired. Owner-supplied mocks require both a question paper and markscheme before publication.
- Every import/upload starts as draft; no upload, cleanup, publication, or withdrawal is scheduled or automatic.
- Do not link/push a remote Supabase project, set remote secrets, deploy functions, create the owner Auth user, or enable any recurring automation without a fresh owner confirmation.
- Stage only the files named by each task; preserve unrelated dirty files.

---

## File Structure

```text
supabase/
  config.toml
  templates/magic_link.html
  migrations/202607120001_private_schema_and_types.sql
  migrations/202607120002_tables_and_constraints.sql
  migrations/202607120003_triggers_and_audit.sql
  migrations/202607120004_authorization_and_rls.sql
  migrations/202607120005_public_rpcs.sql
  migrations/202607120006_owner_rpcs.sql
  migrations/202607120007_private_storage.sql
  migrations/202607120008_takedown.sql
  tests/database/{010_authorization,020_publication,030_distribution,040_storage,050_audit_takedown}.test.sql
  functions/
    _shared/{cors,errors,clients,admin,validation,rateLimit}.ts
    owner-pdf-upload-ticket/{index,handler.test}.ts
    owner-finalize-pdf/{index,handler.test}.ts
    public-document-url/{index,handler.test}.ts
    submit-takedown/{index,handler.test}.ts
    owner-delete-document/{index,handler.test}.ts
scripts/{generate-database-types,import-knowledge}.ts
src/shared/api/{supabaseClient,database.types,publicContent.supabase,ownerAuth.supabase,ownerContent.supabase}.ts
src/features/owner-auth/{OwnerGate,OtpRequestForm,OtpVerifyForm}.tsx
src/layouts/OwnerShell.tsx
src/features/owner-papers/{OwnerPastPapersPage,PastPaperEditor,DocumentPairEditor,QuestionIndexEditor}.tsx
src/features/owner-mocks/{OwnerMocksPage,OwnerMockEditor}.tsx
src/features/legacy-import/{legacyPaperStore,LegacyPaperImportPanel}.ts(x)
src/features/takedown/TakedownPage.tsx
```

### Task 1: Initialize local Supabase and safe environment handling

**Files:**
- Modify: `package.json`
- Modify: `package-lock.json`
- Create: `supabase/config.toml`
- Create: `supabase/templates/magic_link.html`
- Create: `.env.example`
- Modify: `.gitignore`
- Create: `src/shared/api/supabaseClient.ts`
- Create: `src/shared/components/TurnstileWidget.tsx`
- Test: `src/shared/api/supabaseClient.test.ts`
- Test: `src/shared/components/TurnstileWidget.test.tsx`

**Interfaces:**
- Produces `getSupabaseClient(): SupabaseClient<Database>` and `getPublicEnv()`.
- Browser config contains no owner email or service-role key.

- [ ] **Step 1: Write failing environment tests**

```ts
import { readFileSync } from "node:fs";

it("accepts only the public browser variables", () => {
  expect(getPublicEnv({
    VITE_SUPABASE_URL: "https://example.supabase.co",
    VITE_SUPABASE_PUBLISHABLE_KEY: "sb_publishable_test",
    VITE_TURNSTILE_SITE_KEY: "site-key",
  })).toEqual({
    supabaseUrl: "https://example.supabase.co",
    publishableKey: "sb_publishable_test",
    captchaSiteKey: "site-key",
  });
});

it("does not define secret or owner-email variables", () => {
  const text = readFileSync(".env.example", "utf8");
  expect(text).not.toMatch(/SERVICE_ROLE|OWNER_EMAIL|TURNSTILE_SECRET/i);
});
```

- [ ] **Step 2: Run the test to verify failure**

Run: `npm test -- src/shared/api/supabaseClient.test.ts`

Expected: FAIL because the module does not exist.

- [ ] **Step 3: Install CLI and configure local Auth**

Run:

```bash
NPM_CONFIG_CACHE=/tmp/npm-cache npm install --save-exact @marsidev/react-turnstile@1.5.3
NPM_CONFIG_CACHE=/tmp/npm-cache npm install --save-dev --save-exact supabase@2.109.1
```

Set local Auth signup/email signup false, OTP length `6`, expiry `600` seconds, and resend frequency `1m`. The email template renders `{{ .Token }}` as the primary action and never embeds an owner address. `.env.example` contains exactly the three public variables tested above.

`getPublicEnv()` validates with Zod and returns an unavailable-backend result for missing development values; production startup renders a generic configuration error rather than logging secrets. `TurnstileWidget` returns a short-lived token through `onToken(token)` and clears it after use, error, or expiry.

- [ ] **Step 4: Run tests and commit**

Run: `npm test -- src/shared/api/supabaseClient.test.ts src/shared/components/TurnstileWidget.test.tsx`

Expected: PASS.

```bash
git add package.json package-lock.json supabase/config.toml supabase/templates/magic_link.html .env.example .gitignore src/shared/api/supabaseClient.ts src/shared/api/supabaseClient.test.ts src/shared/components/TurnstileWidget.tsx src/shared/components/TurnstileWidget.test.tsx
git commit -m "chore: configure local Supabase safely"
```

### Task 2: Create private schema, enums, content, paper, document, review, and audit tables

**Files:**
- Create: `supabase/migrations/202607120001_private_schema_and_types.sql`
- Create: `supabase/migrations/202607120002_tables_and_constraints.sql`
- Create: `supabase/migrations/202607120003_triggers_and_audit.sql`
- Test: `supabase/tests/database/001_schema.test.sql`

**Interfaces:**
- Enums: `course_code`, `publication_status`, `index_status`, `document_role`, `upload_status`, `cleanup_status`, `calculator_mode`, `topic_tag_role`, `takedown_status`, `pairing_requirement`.
- Stable tables implement every data-model record in the approved specification plus `knowledge_releases` and private takedown/rate-limit state.

- [ ] **Step 1: Write the failing pgTAP schema contract**

```sql
begin;
select plan(9);
select has_schema('private');
select has_table('private', 'admin_users');
select has_table('private', 'topics');
select has_table('private', 'knowledge_releases');
select has_table('private', 'past_papers');
select has_table('private', 'past_paper_documents');
select has_table('private', 'past_paper_questions');
select has_table('private', 'mock_documents');
select has_table('private', 'audit_events');
select * from finish();
rollback;
```

- [ ] **Step 2: Run the local reset/test and verify failure**

Run: `npx supabase start && npx supabase db reset && npx supabase test db`

Expected: schema assertions FAIL before migrations exist.

- [ ] **Step 3: Implement exact schema rules**

Create the non-exposed `private` schema. Tables include syllabus versions/statements, courses/domains/topics/prerequisites/topic mappings, `knowledge_releases`, questions/assets/reviews, Paper 3 tasks, mock sets/papers/questions/documents, past papers/documents/questions/question topics/paper topics, `admin_users`, immutable `audit_events`, `takedown_requests`, and `edge_rate_limits`. A knowledge release stores course/version, generated-source checksum, coverage-report checksum, publication state, reviewer evidence, and publish actor/time.

Past-paper constraints include:

```sql
check (byte_count between 1 and 52428800),
check (mime_type = 'application/pdf'),
check (sha256 ~ '^[0-9a-f]{64}$'),
unique nulls not distinct (past_paper_id, document_role, deleted_at)
```

Use a partial unique index for one primary tag per indexed question and a unique `(past_paper_id, question_code)`. Rights basis, confirmation time, and confirmer are either all present or all absent. `pairing_requirement` is `one_or_both` for past papers and `question_and_markscheme` for mocks. Triggers update timestamps, reject cross-course topic links, mark an edited question index `in_progress`, and forbid update/delete on audit rows.

- [ ] **Step 4: Run tests and commit**

Run: `npx supabase db reset && npx supabase test db`

Expected: all schema assertions PASS.

```bash
git add supabase/migrations/202607120001_private_schema_and_types.sql supabase/migrations/202607120002_tables_and_constraints.sql supabase/migrations/202607120003_triggers_and_audit.sql supabase/tests/database/001_schema.test.sql
git commit -m "feat: add private content and document schema"
```

### Task 3: Deny base tables and add hardened authorization/public RPCs

**Files:**
- Create: `supabase/migrations/202607120004_authorization_and_rls.sql`
- Create: `supabase/migrations/202607120005_public_rpcs.sql`
- Test: `supabase/tests/database/010_authorization.test.sql`
- Test: `supabase/tests/database/030_distribution.test.sql`

**Interfaces:**
- Public RPCs: `is_admin()`, `get_public_availability(p_course course_code)`, `list_published_topics(p_course course_code)`, `get_published_topic(p_course course_code,p_topic_id text)`, `list_published_past_papers(p_course,p_year_from,p_year_to,p_sessions,p_timezones,p_papers)`, `get_published_past_paper(p_paper_id uuid)`, and `get_exam_distribution(p_course,p_year_from,p_year_to,p_sessions,p_timezones,p_papers,p_calculator_modes,p_domain_ids,p_topic_ids)`.
- No RPC exposes object path, bucket, rights notes, owner UUID, original filename, internal review data, requester identity, or raw IP.

- [ ] **Step 1: Write failing authorization and distribution tests**

Tests impersonate `anon`, a non-admin authenticated user, and an admin. Assert anon/non-admin cannot use `private`, read any base table, mutate records, or call admin RPCs; `admin_users` remains unreadable. Assert public RPCs return only published rows and safe keys. Distribution excludes incomplete indexes, counts marks only once through the primary tag, emits canonical topic pairs, and reports both published/indexed paper counts and covered sessions.

- [ ] **Step 2: Run tests to verify failure**

Run: `npx supabase db reset && npx supabase test db`

Expected: FAIL because hardened functions do not exist.

- [ ] **Step 3: Implement the admin check and grants**

```sql
create or replace function public.is_admin()
returns boolean
language sql stable security definer
set search_path = ''
as $$
  select exists (
    select 1 from private.admin_users
    where user_id = auth.uid() and disabled_at is null
  )
$$;
revoke all on function public.is_admin() from public;
grant execute on function public.is_admin() to authenticated;
```

Revoke schema usage/base-table privileges from `anon` and `authenticated`, enable RLS on every private table, and add defense-in-depth admin policies. Owner UI still uses audited RPCs, not direct tables. Every `SECURITY DEFINER` function uses `set search_path=''`, fully qualified references, explicit argument types, explicit revoke/grant, and a private `require_admin()` guard.

`get_exam_distribution()` returns exact keys `coverage`, `totals`, `byDomain`, `byTopic`, `bySession`, `byPaper`, and `cooccurrences`. The denominator applies paper-level filters; only published/complete indexes contribute questions; topic/calculator filters operate at question level.

- [ ] **Step 4: Run tests and commit**

Run: `npx supabase db reset && npx supabase test db`

Expected: authorization and distribution tests PASS.

```bash
git add supabase/migrations/202607120004_authorization_and_rls.sql supabase/migrations/202607120005_public_rpcs.sql supabase/tests/database/010_authorization.test.sql supabase/tests/database/030_distribution.test.sql
git commit -m "feat: expose safe public data RPCs"
```

### Task 4: Add audited owner CRUD, publication gates, storage bucket, and knowledge import

**Files:**
- Create: `supabase/migrations/202607120006_owner_rpcs.sql`
- Create: `supabase/migrations/202607120007_private_storage.sql`
- Create: `scripts/import-knowledge.ts`
- Test: `supabase/tests/database/020_publication.test.sql`
- Test: `supabase/tests/database/040_storage.test.sql`

**Interfaces:**
- Owner RPCs: `admin_list_dashboard()`, `admin_publish_knowledge_release(p_course,p_version_id,p_source_checksum,p_coverage_checksum)`, `admin_withdraw_knowledge_release(p_course,p_version_id,p_reason)`, `admin_create_past_paper(p_payload)`, `admin_update_past_paper(p_id,p_payload)`, `admin_replace_past_paper_index(p_id,p_manifest)`, `admin_complete_past_paper_index(p_id)`, `admin_publish_past_paper(p_id)`, `admin_withdraw_past_paper(p_id,p_reason)`, `admin_create_mock_draft(p_payload)`, `admin_update_mock(p_id,p_payload)`, `admin_publish_mock(p_id)`, `admin_withdraw_mock(p_id,p_reason)`, and `admin_list_review_queue()`.
- Service-only RPCs: `internal_prepare_pdf_upload`, `internal_finalize_pdf_upload`, `internal_resolve_published_document`, `internal_begin_delete`, `internal_finish_delete`, `internal_mark_cleanup_failed`, `internal_consume_rate_limit`, `internal_submit_takedown`.
- `scripts/import-knowledge.ts` accepts a validated generated index and upserts deterministic draft records; it never publishes.

- [ ] **Step 1: Write failing publication/storage tests**

Assert missing rights, pending/invalid documents, wrong role, invalid checksum, and cleanup state block publication. Past papers accept one ready role; mocks require both. Withdrawal removes public visibility. Storage is private; anon/non-admin operations fail; admin paths match `^(past-papers|mock-papers)/<uuid>/<uuid>\.pdf$`.

- [ ] **Step 2: Run tests and verify failure**

Run: `npx supabase db reset && npx supabase test db`

Expected: FAIL before RPCs/bucket exist.

- [ ] **Step 3: Implement transactional metadata mutations**

Every mutation calls `require_admin()`, validates payload keys/types, changes metadata, and writes one immutable audit row in the same transaction. Knowledge-release publication requires matching source/coverage checksums, 100% statement coverage, zero graph/content warnings, and recorded syllabus plus solution-review evidence. Index completion requires every question to have exactly one primary topic. Paper publish checks rights and document rules. Delete first withdraws/marks cleanup; storage deletion is performed outside the DB transaction and finalized or marked retryable.

Create `private-pdfs` with `public=false`, size limit `52428800`, and allowed MIME `application/pdf`. No anon select policy exists. Admin storage policies include both `USING` and `WITH CHECK` and path validation.

- [ ] **Step 4: Validate draft-only content import**

Run: `npm run content:validate && npm run content:build && npx tsx scripts/import-knowledge.ts --dry-run content/generated/knowledge-index.json`

Expected: deterministic insert/update counts, zero deletes, all target statuses `draft`, and no service key in output.

- [ ] **Step 5: Run tests and commit**

Run: `npx supabase db reset && npx supabase test db`

Expected: PASS.

```bash
git add supabase/migrations/202607120006_owner_rpcs.sql supabase/migrations/202607120007_private_storage.sql scripts/import-knowledge.ts supabase/tests/database/020_publication.test.sql supabase/tests/database/040_storage.test.sql
git commit -m "feat: add audited owner publication workflow"
```

### Task 5: Implement six-digit OTP and the admin route gate

**Files:**
- Create: `src/domain/owner.ts`
- Create: `src/shared/api/ownerAuth.supabase.ts`
- Create: `src/features/owner-auth/OwnerGate.tsx`
- Create: `src/features/owner-auth/OtpRequestForm.tsx`
- Create: `src/features/owner-auth/OtpVerifyForm.tsx`
- Create: `src/layouts/OwnerShell.tsx`
- Modify: `src/app/createRoutes.tsx`
- Test: `src/shared/api/ownerAuth.supabase.test.ts`
- Test: `src/features/owner-auth/OwnerGate.test.tsx`

**Interfaces:**

```ts
export type OwnerAccess = "signed_out" | "authenticated_non_admin" | "admin";
export interface OwnerAuthRepository {
  getAccess(): Promise<OwnerAccess>;
  requestOtp(input: { email: string; captchaToken: string }): Promise<void>;
  verifyOtp(input: { email: string; token: string }): Promise<OwnerAccess>;
  signOut(): Promise<void>;
}
```

- [ ] **Step 1: Write failing auth tests**

Assert `signInWithOtp` receives `shouldCreateUser:false` and the CAPTCHA token; token accepts exactly six digits; send UI always says “If this address is authorized, a code has been sent.”; email exists only in component memory; non-admin sees a generic denial; refresh rechecks session/admin status; public shell has no owner link.

- [ ] **Step 2: Run tests to verify failure**

Run: `npm test -- src/shared/api/ownerAuth.supabase.test.ts src/features/owner-auth/OwnerGate.test.tsx`

Expected: FAIL because auth modules do not exist.

- [ ] **Step 3: Implement exact OTP call**

```ts
await client.auth.signInWithOtp({
  email,
  options: { shouldCreateUser: false, captchaToken },
});
```

Verify with `verifyOtp({ email, token, type: "email" })`, then call `is_admin()`. Never log Supabase error details with the address. Show cooldown/expiry/network states in English without account-existence disclosure. Lazy-load owner routes outside the public bundle/layout.

- [ ] **Step 4: Run tests and commit**

Run: `npm test -- src/shared/api/ownerAuth.supabase.test.ts src/features/owner-auth/OwnerGate.test.tsx`

Expected: PASS.

```bash
git add src/domain/owner.ts src/shared/api/ownerAuth.supabase.ts src/features/owner-auth src/layouts/OwnerShell.tsx src/app/createRoutes.tsx
git commit -m "feat: gate owner area with email OTP"
```

### Task 6: Implement two-phase verified PDF upload and 60-second public URLs

**Files:**
- Create: `supabase/functions/_shared/{cors,errors,clients,admin,validation}.ts`
- Create: `supabase/functions/owner-pdf-upload-ticket/{index.ts,handler.test.ts}`
- Create: `supabase/functions/owner-finalize-pdf/{index.ts,handler.test.ts}`
- Create: `supabase/functions/public-document-url/{index.ts,handler.test.ts}`
- Create: `src/shared/validation/pdf.ts`
- Test: `src/shared/validation/pdf.test.ts`

**Interfaces:**

```ts
type UploadTicketRequest = {
  parentType: "past_paper" | "mock_paper";
  parentId: string;
  role: "question_paper" | "markscheme";
  fileName: string;
  mimeType: "application/pdf";
  sizeBytes: number;
  sha256: string;
  rightsConfirmed: true;
  rightsBasis: string;
};
type UploadTicketResponse = { documentId: string; upload: { path: string; token: string } };
type PublicUrlResponse = { signedUrl: string; expiresIn: 60 };
```

- [ ] **Step 1: Write failing function/client validation tests**

Cover non-admin 403, file over 50 MB, wrong MIME/extension/hash, unconfirmed rights, wrong `%PDF-` bytes, stored-size/hash mismatch, invalid-object removal, draft/withdrawn/nonexistent identical 404, no path leakage, TTL exactly 60, and `Cache-Control:no-store`.

- [ ] **Step 2: Run tests to verify failure**

Run: `deno test supabase/functions --allow-env --allow-net=127.0.0.1 && npm test -- src/shared/validation/pdf.test.ts`

Expected: FAIL before handlers exist.

- [ ] **Step 3: Implement ticket/finalize/resolver handlers**

Ticket validates the real Auth JWT and calls `is_admin`; decoded claims alone are insufficient. It creates pending metadata before returning a signed upload token. Finalize streams/incrementally hashes the private object, validates size/MIME/hash/signature, marks ready, or deletes/marks invalid. Public resolver accepts document UUID only, rechecks published parent/ready document/no cleanup, signs for exactly 60 seconds, and returns indistinguishable 404s.

- [ ] **Step 4: Run tests and commit**

Run: `deno test supabase/functions --allow-env --allow-net=127.0.0.1 && npm test -- src/shared/validation/pdf.test.ts`

Expected: PASS.

```bash
git add supabase/functions/_shared supabase/functions/owner-pdf-upload-ticket supabase/functions/owner-finalize-pdf supabase/functions/public-document-url src/shared/validation/pdf.ts src/shared/validation/pdf.test.ts
git commit -m "feat: verify private PDF uploads and delivery"
```

### Task 7: Build owner past-paper indexing and mock-upload workflows

**Files:**
- Create: `src/shared/api/ownerContent.supabase.ts`
- Create: `src/features/owner-papers/{OwnerPastPapersPage,PastPaperEditor,DocumentPairEditor,QuestionIndexEditor}.tsx`
- Create: `src/features/owner-mocks/{OwnerMocksPage,OwnerMockEditor}.tsx`
- Create: `src/pages/OwnerDashboardPage.tsx`
- Modify: `src/app/createRoutes.tsx`
- Test: `src/features/owner-papers/*.test.tsx`
- Test: `src/features/owner-mocks/*.test.tsx`

**Interfaces:**

```ts
export interface OwnerContentRepository {
  createPastPaperDraft(input: PastPaperDraftInput): Promise<string>;
  uploadDocument(input: VerifiedDocumentInput): Promise<void>;
  replaceQuestionIndex(paperId: string, rows: PastPaperQuestionIndexRow[]): Promise<void>;
  completeQuestionIndex(paperId: string): Promise<void>;
  publishPastPaper(paperId: string): Promise<void>;
  withdrawPastPaper(paperId: string, reason: string): Promise<void>;
  createMockDraft(input: OwnerMockDraftInput): Promise<string>;
  publishMock(mockId: string): Promise<void>;
}
```

- [ ] **Step 1: Write failing workflow tests**

Assert draft-first behavior, required rights basis, course/year/session/timezone/paper/calculator/marks validation, question-code uniqueness, page/mark ranges, exactly one primary topic per indexed question, optional secondary tags, paired document display, mock two-document requirement, preview-before-publish, withdraw, and English errors.

- [ ] **Step 2: Run tests to verify failure**

Run: `npm test -- src/features/owner-papers src/features/owner-mocks`

Expected: FAIL because editors do not exist.

- [ ] **Step 3: Implement forms and RPC adapter**

Use React Hook Form with Zod. Compute SHA-256 before requesting the upload ticket, upload to the returned signed path, finalize, then refresh document status. CSV/JSON index imports parse into the same row schema as manual entry and replace the index transactionally. Buttons disable during mutations; failed cleanup remains visible/retryable; no action auto-publishes.

- [ ] **Step 4: Run tests and commit**

Run: `npm test -- src/features/owner-papers src/features/owner-mocks`

Expected: PASS.

```bash
git add src/shared/api/ownerContent.supabase.ts src/features/owner-papers src/features/owner-mocks src/pages/OwnerDashboardPage.tsx src/app/createRoutes.tsx
git commit -m "feat: add owner paper administration"
```

### Task 8: Preserve and import legacy browser PDFs without deletion

**Files:**
- Create: `src/features/legacy-import/legacyPaperStore.ts`
- Create: `src/features/legacy-import/LegacyPaperImportPanel.tsx`
- Test: `src/features/legacy-import/legacyPaperStore.test.ts`
- Test: `src/features/legacy-import/LegacyPaperImportPanel.test.tsx`

**Interfaces:**
- Reads database `ib-aa-hl-library`, version `1`, object store `papers` without schema upgrade/deletion.
- Import identity is legacy ID plus SHA-256; destination is always draft.

- [ ] **Step 1: Write a failing IndexedDB characterization/import test**

Use `fake-indexeddb` to recreate the existing v1 record shape. Assert the `File` remains readable, legacy DB/store/records remain after import, admin and rights data are required, repeat import is idempotent, and no record publishes automatically.

- [ ] **Step 2: Run tests to verify failure**

Run: `npm test -- src/features/legacy-import`

Expected: FAIL because the adapter/panel do not exist.

- [ ] **Step 3: Implement read-only migration UI behind OwnerGate**

Never call `indexedDB.deleteDatabase`, open a newer version, clear/delete the store, or delete `ib-aa-progress`. Let the owner add missing course/document role/rights fields and explicitly import each item. Keep the current Netlify hostname because IndexedDB is origin-bound.

- [ ] **Step 4: Run tests and commit**

Run: `npm test -- src/features/legacy-import`

Expected: PASS.

```bash
git add src/features/legacy-import
git commit -m "feat: preserve and import legacy local PDFs"
```

### Task 9: Add private takedown intake, owner-triggered deletion, and backend release gates

**Files:**
- Create: `supabase/migrations/202607120008_takedown.sql`
- Create: `supabase/functions/_shared/rateLimit.ts`
- Create: `supabase/functions/submit-takedown/{index.ts,handler.test.ts}`
- Create: `supabase/functions/owner-delete-document/{index.ts,handler.test.ts}`
- Create: `src/features/takedown/TakedownPage.tsx`
- Test: `supabase/tests/database/050_audit_takedown.test.sql`
- Test: `src/features/takedown/TakedownPage.test.tsx`
- Modify: `src/app/createRoutes.tsx`

**Interfaces:**
- Takedown request accepts optional paper/document ID, requester identity privately, relationship, reason code, details, optional HTTPS evidence URL, CAPTCHA token, and honeypot.
- Deletion unpublishes first, deletes storage second, then finalizes or records retryable failure.

- [ ] **Step 1: Write failing abuse/deletion tests**

Assert invalid CAPTCHA/honeypot/rate limit fail, raw IP is never stored, requester data is absent from public RPCs, submission never withdraws content, audit rows cannot mutate, deletion failure leaves content non-public and cleanup retryable, and success removes metadata access/object.

- [ ] **Step 2: Run tests to verify failure**

Run: `npx supabase db reset && npx supabase test db && deno test supabase/functions --allow-env --allow-net=127.0.0.1 && npm test -- src/features/takedown`

Expected: FAIL before implementation.

- [ ] **Step 3: Implement private intake and manual cleanup**

Verify CAPTCHA server-side, HMAC/pepper IP rate-limit keys without raw IP, validate bounded fields, store privately, and return generic `202`. Deletion uses `internal_begin_delete`, storage deletion, then finish/failure RPC; no scheduled retry exists.

- [ ] **Step 4: Run the complete local backend gate**

```bash
npx supabase db reset
npx supabase test db
deno test supabase/functions --allow-env --allow-net=127.0.0.1
npm run typecheck
npm test
npm run build
```

Expected: all exit `0`; no skipped/only tests; anon/non-admin isolation, private bucket, rights gates, audit immutability, and signed URL expiry all pass.

- [ ] **Step 5: Commit**

```bash
git add supabase/migrations/202607120008_takedown.sql supabase/functions/_shared/rateLimit.ts supabase/functions/submit-takedown supabase/functions/owner-delete-document supabase/tests/database/050_audit_takedown.test.sql src/features/takedown src/app/createRoutes.tsx
git commit -m "feat: add private rights and deletion workflows"
```

### Manual remote checkpoint — requires fresh owner confirmation

After all local gates pass, stop. Present the exact proposed remote commands and configuration values without secrets. Only after the owner confirms may an executor:

1. Link the chosen Supabase project.
2. Apply migrations.
3. Configure CAPTCHA and the six-digit email template.
4. Set Edge Function secrets (`SUPABASE_SERVICE_ROLE_KEY`, CAPTCHA secret, rate-limit pepper, allowed origins).
5. Deploy the five Edge Functions.
6. Create/invite the owner Auth user and manually insert that UUID into `private.admin_users`.
7. Import validated knowledge content as draft.

Do not enable continuous deployment, scheduled cleanup, batch publication, or account self-registration.
