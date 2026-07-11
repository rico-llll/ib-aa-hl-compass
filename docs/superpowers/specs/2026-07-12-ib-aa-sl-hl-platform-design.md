# IB AA SL / HL Learning Platform — Design Specification

Date: 2026-07-12  
Status: Draft written specification awaiting owner review

## 1. Product goal

Build one public, English-language self-study website for IB Mathematics: Analysis and Approaches at both SL and HL. Students can switch courses from the global header, follow a complete prerequisite-aware knowledge map, practise a large original question bank, view owner-uploaded past-paper PDFs, and use complete mock papers. Public visitors have read-only access. A separate owner area supports email one-time-password login and is the only place where files or records can be created, edited, published, withdrawn, or deleted.

The product must feel like a serious long-term study system, not a prototype populated with demo cards.

## 2. Non-negotiable requirements

- All public interface copy is English.
- The owner interface and validation messages are also English.
- One site contains both `IB AA SL` and `IB AA HL` with a persistent course switch.
- The public interface contains no upload, edit, delete, or owner-login control.
- The owner area lives at `/owner` and uses a six-digit email OTP.
- Authentication alone is insufficient: write operations require an explicit admin role enforced by database and storage policies.
- The owner email is not rendered in the public interface, URL, or client-side configuration.
- The public may open complete published past-paper PDFs in an inline viewer.
- No official IB paper is preloaded by the implementation. Past papers enter the system only through owner upload.
- Upload requires a rights-confirmation checkbox before a PDF can be published.
- Original questions must remain inside the relevant SL or HL syllabus. Difficulty must never be created by importing out-of-syllabus mathematics.
- Official questions must not be copied, lightly paraphrased, parameter-swapped, or stitched together.
- No Matrix content. Complex locus is not treated as a frequent core pattern. Sequences are not over-weighted.
- AA Statistics and Probability contains no Hypothesis Testing.
- Bayes has exactly five accessible questions and no hard questions.
- Probability density function integration is a substantial part of AA HL Topic 4.
- AA HL practice is difficult overall; AA SL practice is moderate and gains challenge mainly through valid topic combinations, interpretation, and computation.
- No generated or imported record is published automatically. Batch generation, scheduled jobs, continuous deployment, and other recurring automation remain disabled until the owner separately confirms each automation class.

## 3. Platform architecture

### 3.1 Front end and hosting

- React + TypeScript + Vite application.
- Source repository: `rico-llll/ib-aa-hl-compass`.
- Public hosting: the claimed Netlify project. Its public hostname must not contain the owner's name, email address, or GitHub username.
- Manual Netlify releases build from the GitHub repository with `npm run build` and publish `dist`. Continuous deployment is not enabled without separate owner confirmation.
- Public routes are client-side routes with a Netlify SPA fallback.

### 3.2 Backend

Use one Supabase project for:

- Email OTP authentication.
- Postgres content and metadata.
- Private PDF storage with controlled delivery of published documents.
- Row-level security for public-read/admin-write access.

The browser uses only the Supabase project URL and publishable key. No service-role key is shipped to the browser.

### 3.3 Authorization model

- `anon`: may select only published topics, questions, mock papers, and past-paper metadata. Published PDFs are delivered through short-lived signed URLs; draft object paths never receive a public URL.
- `authenticated non-admin`: receives no write access and cannot enter owner workflows.
- `admin`: may insert, update, publish, unpublish, upload, replace, and delete.
- Admin status is stored in a protected `admin_users` table keyed by Supabase Auth user UUID.
- RLS policies call a locked-down `is_admin()` database function that checks `auth.uid()` against `admin_users`; UI hiding is never treated as authorization.
- Before launch, the owner Auth user is created or invited, and its UUID is manually added to `admin_users` before the first OTP login. OTP requests use `shouldCreateUser: false`, so `/owner` cannot create arbitrary accounts.
- OTP responses are generic, CAPTCHA-protected, and subject to server-side throttling. The entered address is sent only to Supabase Auth for the request; it is not bundled into the app, saved to browser storage, sent to analytics, or placed in a URL.
- Browser roles cannot read or mutate `admin_users`. `is_admin()` is `SECURITY DEFINER`, has a fixed safe `search_path`, and is the only browser-callable admin check.
- Anonymous/authenticated browser roles have no direct base-table access. Public reads use explicitly enumerated safe projections/RPCs that omit owner UUIDs, rights notes, internal review fields, object paths, and other private columns.
- The storage bucket is private. Storage policies allow admin writes, while a server-side signed-URL endpoint returns a 60-second URL only after verifying that the linked metadata record is published. Withdrawal stops new URLs immediately; any already-issued URL expires within 60 seconds.

## 4. Information architecture

### 4.1 Global header

- Product identity.
- `AA SL / AA HL` course switch.
- Public navigation: Home, Knowledge Map, Topic Practice, Past Papers, Exam Distribution, Mock Papers, Study Paths.
- `Paper 3 Lab` appears only in AA HL.
- No owner link appears in the public header or footer.

### 4.2 Public pages

#### Home

- Course-specific progress summary stored locally for anonymous students.
- Continue-learning recommendation based on prerequisites and local mastery.
- Entry cards for Knowledge Map, Topic Practice, Past Papers, Exam Distribution, Mock Papers, and Study Paths.
- AA HL additionally surfaces Paper 3 Lab.

#### Knowledge Map

- Complete course-specific prerequisite graph.
- Five domains: Number and Algebra; Functions; Geometry and Trigonometry; Statistics and Probability; Calculus.
- Every node includes:
  - syllabus-aligned learning objectives;
  - prerequisites and successors;
  - definitions and notation;
  - formulas and conditions of use;
  - derivations when pedagogically appropriate;
  - calculator/non-calculator expectations;
  - common IB-style assessment patterns;
  - common errors and diagnostic checks;
  - linked practice questions and learning path position.
- Learning sequence: Concepts → Foundation Check → Worked Example → Exam-style Practice.

#### Topic Practice

- Filters: course, domain, subtopic, difficulty, paper type, calculator status, marks, estimated time, and completion state.
- Question detail includes full prompt, diagrams where required, expandable solution, markscheme, syllabus tags, marks, expected time, and common wrong approaches.
- Public progress, bookmarks, and completion state remain browser-local in the first release.

#### Past Papers

- Metadata filters: course, year, session, timezone, paper, question paper/markscheme, and topic tags.
- Inline full-PDF viewing for published files.
- Question paper and markscheme can be paired.
- No preloaded official PDFs.

#### Exam Distribution

- Distribution data comes only from owner-indexed questions in published past papers; the site does not reproduce official question text.
- Filters: course, year range, session, timezone, paper, calculator status, domain, and subtopic.
- Views: question count, marks share, frequency by session, Paper 1/Paper 2/Paper 3 mix, and subtopic co-occurrence.
- Every chart displays indexed-paper coverage as `indexed published papers / published papers` and the covered sessions. Partial uploads are never described as the universal IB distribution.
- Papers without a completed question index remain viewable in Past Papers but do not enter frequency calculations.

#### Mock Papers

- Separate SL and HL libraries.
- Complete original papers with generated question-paper PDF, separate markscheme PDF, and an accessible HTML question view.
- PDF files are compiled deterministically from the same reviewed source records as the HTML view.
- Topic-practice questions are not reused in mock papers.

#### Paper 3 Lab (HL only)

- Dedicated library of 40 original investigation/problem-solving tasks.
- Tasks are independent of the 20 Paper 1 + Paper 2 mock sets.
- Each task includes staged prompts, markscheme, technology guidance, expected reasoning path, and generalization/verification requirements.
- The 40 tasks are also assembled as 20 two-task practice papers; each pairing totals 55 marks, allows technology, and targets 1 hour.

#### Study Paths

- Course-specific prerequisite routes.
- Students cannot be recommended a node before its necessary prerequisites.
- Path steps follow Concepts → Foundation Check → Worked Example → Exam-style Practice.

### 4.3 Owner area

The unlinked `/owner` route contains:

- Email entry, send-code action, six-digit OTP verification, and sign-out.
- Explicit authorization check after authentication.
- Course and content-type dashboard.
- Past-paper upload form:
  - PDF file;
  - SL/HL;
  - year, session, timezone, paper;
  - question paper or markscheme;
  - title and source label;
  - topic and question-level tags;
  - marks;
  - paired-document reference;
  - draft/published status;
  - mandatory rights confirmation.
- Past-paper question indexing by form or validated CSV/JSON manifest: question number, section, page range, marks, calculator status, and primary/secondary topic tags.
- Owner-supplied mock upload:
  - original question-paper PDF and markscheme PDF pairing;
  - SL/HL, paper, duration, marks, calculator mode, author/source, and topic balance;
  - originality/rights confirmation, preview, and draft/published status;
  - PDF-only display unless the mock is separately authored in the structured HTML content format.
  - supplemental owner uploads do not count toward the required 20 SL or 20 HL mock sets unless they pass the same structured-content, syllabus, solution, and originality review pipeline.
- Question and mock-paper editors.
- Preview-before-publish.
- Publish, withdraw, replace, edit, and delete actions.
- Clear errors for invalid file type, file size, upload failure, duplicate metadata, missing rights confirmation, expired OTP, and unauthorized account.

## 5. Data model

Core tables:

- `courses`: `aa_sl`, `aa_hl`.
- `syllabus_versions`: governing guide, effective examination sessions, source identifier, and checksum.
- `syllabus_statements`: versioned official-course coverage units used as the syllabus gate.
- `domains`: the five syllabus domains per course.
- `topics`: atomic knowledge nodes and teaching content.
- `topic_syllabus_statements`: auditable mapping from every topic node to its syllabus coverage.
- `topic_prerequisites`: directed prerequisite edges.
- `questions`: original standalone practice questions.
- `question_topics`: primary and secondary syllabus tags.
- `question_assets`: diagrams and other reviewed question media.
- `paper3_tasks`: HL investigation tasks.
- `mock_sets`: set-level metadata.
- `mock_papers`: Paper 1 or Paper 2 within a set.
- `mock_questions`: unique questions belonging only to a mock paper.
- `mock_documents`: owner-uploaded or compiled question-paper/markscheme objects, checksums, pairing, and publication metadata.
- `past_papers`: owner-uploaded PDF metadata.
- `past_paper_documents`: storage object, document role, checksum, MIME type, and pairing metadata for question papers and markschemes.
- `past_paper_questions`: indexed question number, section, page range, marks, and calculator status without official question text.
- `past_paper_question_topics`: primary and secondary topic tags for indexed past-paper questions.
- `past_paper_topics`: paper-level summary tags for uploaded papers.
- `content_reviews`: solution, syllabus, originality, and publication-gate review outcomes.
- `audit_events`: immutable owner action records for upload, edit, publish, withdraw, replace, and delete events.
- `admin_users`: protected owner authorization records.

Common publishing fields: `status`, `syllabus_version`, `published_at`, `created_at`, `updated_at`, `created_by`.

Uploaded-document rights metadata includes `rights_basis`, `rights_confirmed_at`, and `rights_confirmed_by`. Database constraints and publish RPCs enforce that a past paper or owner-supplied mock cannot be published unless rights are confirmed and the required document pairing exists.

Question metadata includes:

- course and domain;
- primary and secondary topic IDs;
- difficulty;
- paper type;
- calculator requirement;
- marks;
- estimated minutes;
- prompt body;
- solution body;
- markscheme body;
- common wrong approaches;
- originality fingerprint fields.

Structured mathematical content uses sanitized Markdown plus TeX rendered with KaTeX. Arbitrary HTML and executable embeds are forbidden. Question assets are limited to validated raster images and sanitized SVG, require alt text, and are served with restrictive content types and Content Security Policy headers.

### 5.1 Content source and publication pipeline

- Knowledge-map lessons, standalone questions, Paper 3 tasks, mock questions, solutions, and markschemes live in versioned structured source files before import.
- A schema validator rejects missing fields, invalid course/topic links, incorrect inventory totals, disallowed syllabus tags, invalid mark totals, and forbidden reuse across libraries.
- A syllabus coverage report must map every source record to one or more `syllabus_statements` and must show complete statement coverage for each finished course map.
- The governing baseline is the IB Mathematics: Analysis and Approaches guide first assessed in 2021, plus official corrections and updates effective for the 2026 examination sessions. Its exact document identifiers and checksums are locked in `syllabus_versions` before content import; a later curriculum change creates a new version rather than silently changing existing records.
- A similarity audit combines exact normalized-text checks, fingerprint matching, and semantic candidate detection. Candidate clusters require human review; automation does not make the final originality decision.
- Deterministic content IDs make imports repeatable without duplicating records.
- Every import lands in `draft`. Only an authorized owner action can publish it.
- Mock question-paper and markscheme PDFs are compiled from reviewed source records after validation, so PDF and HTML content cannot silently diverge.

## 6. Content inventory

### 6.1 AA HL topic practice: 850 original questions

| Domain | Count | Required allocation |
|---|---:|---|
| Topic 1: Number and Algebra | 150 | Sequences 20; Permutations and Combinations 30; Binomial Theorem 30; Proof 20; Complex Numbers 50 |
| Topic 2: Functions | 170 | Function concepts/inverses 25; transformations 20; polynomials 40; rational functions 30; exponential/logarithmic functions 30; integrated function problems 25 |
| Topic 3: Geometry and Trigonometry | 180 | Circular measure 20; identities/equations 55, including 20 additional high-difficulty trigonometry questions; trigonometric models 20; vectors 45; lines/planes 40 |
| Topic 4: Statistics and Probability | 150 | Descriptive/bivariate 20; conditional probability including exactly 5 accessible Bayes questions 20; discrete/binomial distributions 30; continuous random variables and PDF integration 45; normal distribution 25; integrated expectation/interpretation 10 |
| Topic 5: Calculus | 200 | Limits/continuity 20; differentiation 35; applications 35; integration/area/volume 30; integration techniques 30; differential equations 25; series/Maclaurin 25 |

HL questions are difficult overall. The hard tier emphasizes multi-stage reasoning, parameter dependence, inverse problems, proof/justification, and natural cross-topic links, while remaining inside the syllabus.

HL difficulty labels are assessment-load labels, not syllabus levels: `Standard` is still a substantive multi-step HL exam-style task; `Hard` requires an extended technique chain or non-obvious setup; `Challenge` adds sustained synthesis, proof, parameter analysis, or high-mark interpretation. No label permits content beyond AA HL.

Required HL difficulty distribution:

| Domain | Standard | Hard | Challenge | Total |
|---|---:|---:|---:|---:|
| Topic 1 | 35 | 80 | 35 | 150 |
| Topic 2 | 40 | 90 | 40 | 170 |
| Topic 3 | 35 | 95 | 50 | 180 |
| Topic 4 | 55 | 75 | 20 | 150 |
| Topic 5 | 35 | 110 | 55 | 200 |
| **Total** | **200** | **450** | **200** | **850** |

The five Bayes questions are included in Topic 4's Standard allocation. Topic 3's 20 additional high-difficulty trigonometry questions are included in both the 55 identities/equations questions and the 180-question Topic 3 total; they comprise 12 Hard and 8 Challenge questions.

Bayes rules:

- Exactly five questions.
- Foundation or standard difficulty only.
- Each can be solved from conditional probability definitions and supplied information without prior memorization of Bayes' theorem.

Probability density function integration includes normalization, definite-integral probability, piecewise densities, CDF/PDF links, median/quartiles, expectation, variance, unknown parameters, and calculus-probability combinations. At least 30 of the 45 continuous-random-variable questions require students to set up or evaluate a definite integral, and at least 15 are multi-stage Hard or Challenge questions.

### 6.2 AA SL topic practice: 580 original questions

| Domain | Count | Required allocation |
|---|---:|---|
| Topic 1: Number and Algebra | 110 | Numerical representation/approximation 10; sequences and series 30; financial applications 15; exponents and logarithms 20; proof and justification 15; binomial theorem 20 |
| Topic 2: Functions | 120 | Concepts/domain/range 15; linear/quadratic/polynomial functions 25; reciprocal/rational functions 15; exponential/logarithmic functions 20; transformations 15; composite/inverse functions 15; modelling and graphical/algebraic solutions 15 |
| Topic 3: Geometry and Trigonometry | 120 | Measurement and spatial geometry 15; coordinate geometry 15; circular measure 15; trigonometric ratios and sine/cosine rules 20; unit-circle relationships, identities, and equations 25; trigonometric models 10; vectors, scalar product, and lines at SL scope 20 |
| Topic 4: Statistics and Probability | 100 | Data collection and sampling 10; descriptive statistics 15; correlation/regression 15; probability and conditional probability 20; discrete random variables and expectation 15; binomial/normal distributions 15; integrated interpretation 10 |
| Topic 5: Calculus | 130 | Derivative concepts and SL rules 30; tangents/normals 15; applications and optimization 25; antiderivatives and definite integrals 25; area and kinematics 20; mixed calculus interpretation 15 |

SL excludes every AHL-only technique. Difficulty is moderate and arises through valid topic combinations, interpretation, graph/algebra conversion, and reasonable computation length rather than obscure tricks.

Required SL difficulty distribution:

| Domain | Foundation | Standard | Stretch | Total |
|---|---:|---:|---:|---:|
| Topic 1 | 25 | 65 | 20 | 110 |
| Topic 2 | 20 | 75 | 25 | 120 |
| Topic 3 | 20 | 75 | 25 | 120 |
| Topic 4 | 25 | 60 | 15 | 100 |
| Topic 5 | 20 | 80 | 30 | 130 |
| **Total** | **110** | **355** | **115** | **580** |

`Stretch` still means fully SL-syllabus-aligned; it increases topic integration and calculation load, not syllabus level. Across the entire original public library, the only five questions whose essential method is Bayes/reverse conditional probability are the accessible HL Topic Practice questions described in section 6.1. Mock papers and Paper 3 contain none. SL conditional-probability questions do not require or name Bayes' theorem.

### 6.3 Paper 3

- HL only.
- 40 unique investigation tasks assembled into 20 two-task Paper 3 practice papers, each totaling 55 marks and 1 hour.
- Every task develops through multiple stages and includes conjecture, generalization, verification, interpretation, or a comparable investigation arc.
- No task ends after one formula or depends on out-of-syllabus theory.

### 6.4 Complete mock papers

AA HL (20 paired sets, 40 papers):

- 20 Paper 1 papers, 110 marks, 2 hours, no calculator.
- 20 Paper 2 papers, 110 marks, 2 hours, calculator allowed.

AA SL (20 paired sets, 40 papers):

- 20 Paper 1 papers, 80 marks, 90 minutes, no calculator.
- 20 Paper 2 papers, 80 marks, 90 minutes, calculator allowed.

The complete library therefore contains 40 paired sets and 80 papers. All 80 papers are original and each has a separate markscheme. Mock questions are not copied from official papers, reused from topic practice, or reused in another mock paper.

## 7. Originality and assessment-quality controls

Every original question stores and is reviewed against six fingerprint dimensions:

1. Core mathematical object.
2. Required technique chain.
3. Representation.
4. Unknown being solved or statement being justified.
5. Context/abstraction type.
6. Topic combination.

A question is rejected when it merely changes values, names, symbols, surface context, or ordering while preserving the same mathematical structure.

Every question and paper must pass:

- syllabus gate;
- independent solution check;
- markscheme and follow-through check;
- command-term and notation check;
- calculator-mode check;
- mark/time-load check;
- similarity-fingerprint audit;
- course-specific difficulty check;
- whole-paper topic balance and difficulty-curve check.

Additional constraints derived from prior review:

- Calculus receives the greatest HL weight.
- Functions emphasize polynomial and rational structures.
- Complex numbers receive a high share of extended/high-mark questions.
- Statistics leans toward Paper 2.
- Sequences are not overrepresented.
- No matrices.
- Complex locus is not treated as a frequent core pattern.
- Paper questions must develop mathematically; superficial one-formula modelling is rejected.
- Hypothesis Testing is excluded from knowledge-map lessons, topic practice, mock papers, Paper 3, tags, and recommendations for both courses.

### 7.1 Study-path and mastery rules

- A student may browse any knowledge node; `locked` means “not yet recommended,” not access denied.
- `ready` requires all direct prerequisite nodes to be mastered. `learning` begins after the first saved attempt.
- A node becomes `mastered` after an 80% foundation-check score and at least one linked exam-style question completed correctly without revealing the solution before submission. An incomplete attempt does not count as correct or lower the stored score.
- Progress keys include course, syllabus version, and topic ID, so SL/HL state cannot collide.
- Prerequisite data must pass missing-node and cycle validation before publication.
- The recommendation engine chooses the earliest ready, unmastered node in the selected study path; ties are resolved by lowest recent mastery, then syllabus order.

## 8. Error handling and safety

- Public reads fail gracefully with an English retry state and no owner details.
- Unpublished data is never returned by public queries.
- Unauthorized writes fail at RLS even if requests are manually constructed.
- PDF upload validates MIME type, extension, size, and rights confirmation.
- PDF upload accepts a maximum of 50 MB per file and verifies the PDF file signature in addition to extension and declared MIME type.
- Deleting a paper requires explicit confirmation. The system records an audit event, removes public metadata access first, then deletes the storage object; a retryable cleanup state handles partial failure across database and storage.
- Question paper/markscheme pairing is transactional at the metadata level.
- OTP resend is rate-limited by Supabase; the UI shows cooldown and expiration states.
- The public PDF viewer receives an expiring signed URL only after the published-state check and opens it in a sandboxed viewer context where practical.
- Past-paper publication also records the rights basis, confirmation timestamp, and confirming admin. A public non-affiliation/copyright notice and a CAPTCHA-protected, non-personal takedown form are launch requirements; submitted requests enter the owner review queue without exposing the owner's email or automatically withdrawing content.
- Markdown/TeX rendering, asset sanitization, and CSP prevent stored content from executing arbitrary script.

## 9. Validation and acceptance criteria

### Functional

- An unauthenticated visitor can switch SL/HL, browse published data, and open published PDFs.
- An unauthenticated visitor can inspect Exam Distribution and see exactly which indexed paper set supports every chart.
- No public page renders an upload or admin control.
- A non-admin authenticated account cannot write through UI or direct API calls.
- The designated admin can complete OTP login and CRUD workflows.
- Draft content is invisible publicly until published.
- Course switching never leaks SL/HL topics or progress into the other course.

### Content

- All five domains and complete prerequisite maps exist for both courses.
- Every official syllabus statement in the configured syllabus version maps to at least one published knowledge node, and the coverage report has no unresolved gaps or SL-to-AHL leakage.
- Final inventory reaches 850 HL topic questions, 580 SL topic questions, 40 HL Paper 3 tasks, 40 HL mock papers, and 40 SL mock papers.
- Every original practice, mock, and Paper 3 question record has a solution, markscheme, tags, marks, calculator status, expected time, and fingerprint.
- Every knowledge node has objectives, prerequisites, teaching content, notation, at least one worked example, a foundation check, common errors, and syllabus-statement mappings.
- Every published past paper has rights metadata, at least one PDF document, public-safe metadata, and a checksum; question-level indexing is optional but its completion state is explicit.
- Topic counts match the allocations in this specification.
- Bayes/reverse-conditional essential-method count is exactly five across topic practice, mocks, and Paper 3; all five are HL Topic Practice Standard questions.
- Hypothesis Testing count is zero across all authored and indexed public content types; uploaded PDFs are preserved as supplied and are never rewritten.
- HL Topic 3 includes the additional 20 hard trigonometry questions.
- PDF-integration coverage matches section 6.1.
- Automated similarity reports contain no unresolved duplicate clusters before public release.

### Technical

- Type checking and production build pass.
- Unit tests cover prerequisite logic, course isolation, filters, fingerprint comparison, and data validation.
- Integration tests cover OTP state handling, admin authorization, publish/unpublish, PDF upload metadata, and public read policies.
- Security tests cover disabled account creation, generic OTP responses, admin-table isolation, safe public projections, rights-gated publication, signed-URL expiry, Markdown/asset sanitization, and base-table denial.
- Browser QA covers desktop and mobile navigation, the SL/HL switch, public PDF view, and owner workflows.
- Accessibility: keyboard navigation, visible focus, form labels, meaningful status text, and WCAG AA contrast targets.

## 10. Delivery strategy and scope decomposition

This document is the program-level master specification. The product is too large for one safe implementation batch, so implementation and content are delivered as independently reviewed subprojects. Incomplete cards are not presented as finished content, and public navigation does not link to an unfinished library.

1. Platform foundation: English UI, course switch, route structure, Supabase schema/RLS, OTP owner area, public-safe PDF delivery, rights/takedown controls, Exam Distribution structure, and full SL/HL knowledge maps.
2. HL topic bank: 850 reviewed questions.
3. HL Paper 3 Lab: 40 reviewed tasks.
4. SL topic bank: 580 reviewed questions.
5. HL mock library: 20 Paper 1 + Paper 2 sets.
6. SL mock library: 20 Paper 1 + Paper 2 sets.
7. Whole-library similarity, syllabus, solution, markscheme, and paper-balance audit.

Each milestone is publishable only after its own acceptance tests pass. Milestones 2 through 6 each receive a dedicated inventory manifest, implementation plan, and review checkpoint before production. The next implementation plan covers Milestone 1 only; completing the foundation does not imply that the numerical content inventories are already filled. Unfinished libraries and routes stay out of public navigation, and every displayed count is computed only from reviewed, published records.
