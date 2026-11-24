# Change specification: Inactive book discovery

## Metadata

- **Owner:** Unknown
- **Status:** Draft
- **Source root:** `legacy-src\dotnet-bookcatalog`
- **Related issue:** Not created

## Problem and outcomes

### Problem

Users of the BookCatalog UI can mark a book inactive but cannot discover that
book from the catalog to reactivate it. The default route displays only active
books, and its row actions are rendered only for those results. Users who
already know a book's ID can still navigate directly to its Details, Edit,
and Delete routes.

Evidence, relative to the repository root:

- `legacy-src\dotnet-bookcatalog\shared-legacy-app\src\BookCatalog.Web\App_Start\RouteConfig.cs:12-15`, `RouteConfig.RegisterRoutes`, maps the default route to `Books/Index`.
- `legacy-src\dotnet-bookcatalog\shared-legacy-app\src\BookCatalog.Web\Controllers\BooksController.cs:13-19`, `BooksController.Index`, filters to `IsActive` and orders by title.
- `legacy-src\dotnet-bookcatalog\shared-legacy-app\src\BookCatalog.Web\Views\Books\Index.cshtml:29-49` renders Details, Edit, and Delete links only for the rows supplied by Index.
- `legacy-src\dotnet-bookcatalog\shared-legacy-app\src\BookCatalog.Web\Views\Books\Create.cshtml:50` and `Views\Books\Edit.cshtml:51` expose the Active checkbox.
- `legacy-src\dotnet-bookcatalog\shared-legacy-app\src\BookCatalog.Web\Controllers\BooksController.cs:53-82`, `BooksController.Edit`, looks up a book by ID, saves `IsActive` on valid input, and redirects to Index.
- `legacy-src\dotnet-bookcatalog\shared-legacy-app\src\BookCatalog.Web\Models\ApplicationDbContext.cs:8-13` defines the EF6 `Books` set; `src\BookCatalog.Web\Web.config:6-7` configures LocalDB. `Models\ApplicationDbContext.cs:20-25,38` defines the model-change initializer and seeds an inactive book; `Global.asax.cs:14-26` initializes and seeds when the table is empty.
- `legacy-src\dotnet-bookcatalog\shared-legacy-app\BookCatalog.sln` contains only the web project. `legacy-src\dotnet-bookcatalog\AGENTS.md` documents that there is no automated baseline test project.
- `scripts\validation\Test-BookCatalogSmoke.ps1:163-178` tests inactive filtering and reactivation in the modernized target by retaining a known ID. It does not test discovery through the baseline UI and is not baseline validation.

The user confirmed that inactive books may become discoverable to all users who
can access the existing catalog. Whether real deployments contain sensitive
book data, and any host-level authorization surrounding this sample, are
unknown.

### Desired outcomes

1. A user can navigate from the active catalog to a view of inactive books,
   find a book, and reactivate it using the existing Edit form.
2. The default active-only catalog, MVC 5/EF6/.NET Framework 4.8 stack,
   existing database schema, CRUD behavior, validation, antiforgery, and
   direct routes remain compatible.
3. Focused automated checks and the documented Release build and disposable
   LocalDB manual checks provide observable evidence for the behavior.

### Success measures

- Active books remain the only records in `/` and `/Books`; inactive records
  appear in a separately discoverable, inactive-only view.
- A reactivated book appears in the next active catalog request and no longer
  appears in the inactive view.
- Empty results, input boundaries, missing records, existing navigation and
  antiforgery behavior match the acceptance criteria below.
- A new focused MSTest project executes one or more tests; zero discovered or
  executed tests is not a pass.
- The primary Release build, focused automated tests, and required manual
  checks complete successfully. No result is claimed until observed.

## Functional requirements

| ID | Requirement | Evidence or rationale |
| --- | --- | --- |
| FR-1 | The system shall retain the default `/` and `/Books` active-only catalog, ordered by ascending title, and provide navigation from it to the inactive-only view. | `legacy-src\dotnet-bookcatalog\shared-legacy-app\src\BookCatalog.Web\Controllers\BooksController.cs:13-19`; `App_Start\RouteConfig.cs:12-15`; `Views\Books\Index.cshtml:29-49` |
| FR-2 | When the inactive view is requested, the system shall list current inactive books only, including books created inactive and books later deactivated, in ascending title order. | `Models\Book.cs:26`; `Models\ApplicationDbContext.cs:38`; inactive filtering is the identified gap |
| FR-3 | Each inactive row shall offer the existing Details, Edit, and Delete actions. | Existing row actions: `Views\Books\Index.cshtml:49`; direct lookup: `BooksController.cs:25,53-60,89-107` |
| FR-4 | The inactive view shall provide a way to return to the active catalog and, when there are no inactive books, display a clear no-inactive-books message. | Existing list navigation conventions: `Views\Books\Details.cshtml:47-49`; active list empty state: `Views\Books\Index.cshtml` |
| FR-5 | The user shall reactivate an inactive book through the existing Edit form and Active checkbox. After a successful valid edit, the existing redirect to the active catalog shall remain. | `Views\Books\Edit.cshtml:9,51`; `BooksController.cs:65-82` |
| FR-6 | Inactive-list navigation and filtering shall use read-only GET behavior and shall not change book records. | Existing list action is GET and queries `db.Books`: `BooksController.cs:13-19` |
| FR-7 | Direct Details, Edit, and Delete routes shall remain usable for either status; existing Cancel, Details Back, Delete Cancel, and successful Delete navigation shall continue to lead to the active Index. | `BooksController.cs:25,53-60,89-107`; `Views\Books\Edit.cshtml:60`; `Views\Books\Details.cshtml:48`; `Views\Books\Delete.cshtml:50-52` |
| FR-8 | The system shall preserve existing input validation, including required title and author, title/author/ISBN maximum lengths of 200/100/13, and an optional published year in the inclusive range 1800-2100. It shall add no ISBN format rule. | `Models\Book.cs:10-23` |
| FR-9 | The system shall preserve current missing-record behavior: Details, Edit, and Delete GETs and valid Edit POST for a missing ID return 404; invalid Edit POST retains validation precedence; Delete POST for a missing ID retains its redirect to Index. | `BooksController.cs:25-30,53-60,65-83,99-107` |
| FR-10 | The system shall retain antiforgery protection on Edit and Delete POSTs. A missing or invalid token shall not update the book. | `BooksController.cs:63-66,97-100`; `Views\Books\Edit.cshtml:9-10`; `Views\Books\Delete.cshtml:45-47` |
| FR-11 | A query or save failure shall not be represented as a successful save or as a normal empty result; preserve observed baseline error handling without adding automatic retry or a rollback guarantee. | Existing controller has no error-to-empty fallback: `BooksController.cs:13-19,65-83`; failure behavior has not been run or characterized |
| FR-12 | The system shall preserve saved state across application restart against the same disposable database. A stale form that saves after a prior edit shall retain the existing no-concurrency-token behavior; no outcome is promised for a delete/save race. | `Models\Book.cs:6-35` has no concurrency token; edit copies submitted fields at `BooksController.cs:73-80`; restart persistence is a documented baseline check |
| FR-13 | The implementation shall add a focused .NET Framework 4.8 MSTest project that checks actual application filtering, ordering, and empty-list behavior deterministically, without requiring IIS, LocalDB, Azure, or network access after dependencies are restored. | No baseline test project: `legacy-src\dotnet-bookcatalog\AGENTS.md`; project uses .NET Framework 4.8: `src\BookCatalog.Web\BookCatalog.Web.csproj` |

## Constraints and out of scope

### Constraints

- **Allowed paths for implementation source changes:**
    - `legacy-src\dotnet-bookcatalog\shared-legacy-app\src\BookCatalog.Web\Controllers\`
    - `legacy-src\dotnet-bookcatalog\shared-legacy-app\src\BookCatalog.Web\Views\Books\`
    - `legacy-src\dotnet-bookcatalog\shared-legacy-app\src\BookCatalog.Web\BookCatalog.Web.csproj`
    - `legacy-src\dotnet-bookcatalog\shared-legacy-app\BookCatalog.sln`
    - `legacy-src\dotnet-bookcatalog\shared-legacy-app\tests\BookCatalog.Web.Tests\`
- The specification artifact itself is `specs\inactive-book-discovery.md`; it is not permission to change other paths. Directory allowances above include descendants; all other source paths are out of scope unless separately approved.
- **Primary validation command:** Run from `legacy-src\dotnet-bookcatalog` with Visual Studio MSBuild:

  ```powershell
  msbuild .\shared-legacy-app\BookCatalog.sln -restore -p:Configuration=Release
  ```

  This is the documented classic Release build, not a behavior test. The command
  has not been run for this change. A successful build alone is not a full pass.
- **Additional required validation:** Run the new MSTest project with the
  available Visual Studio test runner, recording the exact command, number of
  executed tests, and result. The runner, adapter, package versions, and exact
  test command are unknown until checked in the Windows toolchain; do not
  invent them. Perform the documented IIS Express/LocalDB checks in
  `labs\04-app-modernization\dotnet\01-assess-decide-plan.md:89-105`, extended
  for this specification. Use synthetic data and a verified task-owned
  disposable database. Do not treat the modernized-target smoke script as
  baseline validation.
- **Compatibility:** Preserve .NET Framework 4.8, classic MVC 5, EF6,
  `packages.config`, solution layout, schema, attribution, existing access
  behavior, CRUD, direct URLs, redirects, validation, antiforgery, and error
  behavior. Add no new concurrency control. Preserve the active-only default.
- **Dependencies:** Use MSTest as selected by the user. Package versions and
  compatible Visual Studio test adapter/runner are unknown. Add only the
  minimum test dependencies needed; if compatibility or dependency approval
  cannot be established within allowed paths, stop.
- **Data and security:** Use only synthetic fixtures and a disposable,
  task-owned LocalDB database. Never connect to a business database or use
  secrets or real personal, participant, customer, or tenant data. Preserve
  Razor's normal text encoding; do not render book fields as raw HTML. Do not
  collect or log record contents, credentials, connection strings, or request
  tokens. `src\BookCatalog.Web\Web.config:18` sets `customErrors` to `Off`;
  do not claim current error pages hide technical details or alter global
  error configuration within this scope.
- **Generated files and data:** Restore/build may create generated
  `packages`, `bin`, or `obj` outputs; do not hand-edit or commit them. Do not
  modify or clean a database until its disposable ownership is verified.
  `Models\ApplicationDbContext.cs:20-25` uses a drop/create-on-model-change
  initializer and `Global.asax.cs:14-26` initializes/seeds on startup, so
  establish a safe baseline and avoid a zero-row table for the empty-inactive
  check: startup seeds an entirely empty table.

### Exclusions (out of scope)

- Changing the default active-only view or showing both statuses by default.
- A dedicated Reactivate action; use the existing Edit checkbox.
- Restricting or expanding access beyond current application behavior. The user
  accepted inactive-book visibility for existing catalog users; host-level
  authorization is not verified.
- Framework, runtime, ORM, or database-schema upgrades; migrations; changes to
  the Java track, capstone applications, automation, Azure examples, or
  repository-wide configuration.
- Search, pagination, a combined-status view, a new ISBN format rule, or a new
  ordering guarantee for equal titles.
- Concurrency controls, automatic retry, a new error experience, or changes
  to unrelated CRUD/startup behavior.
- Editing generated files, deleting unowned data, or using an empty MDF as a
  startup workaround.

## Edge cases

| Type | Condition | Expected behavior |
| --- | --- | --- |
| Normal | Active and inactive books exist; request `/` or `/Books`. | Only active books appear in ascending title order; navigation to the inactive view is available. |
| Normal | Inactive books include one created inactive and one later deactivated. | Both appear only in the inactive view, ordered by title, with existing Details, Edit, and Delete actions. |
| Normal | A user selects Active on Edit for an inactive book and submits valid data with a valid token. | Existing save succeeds, redirects to active Index, and subsequent requests reflect the new status. |
| Failure | An inactive-list query fails. | Do not render a normal empty state; use observed existing error handling. |
| Failure | An Edit save fails. | Do not claim a successful save or automatically retry; preserve observed error handling and separately check persisted state where safe. Do not assume rollback. |
| Failure | A GET, valid Edit POST, or other action addresses a missing book ID. | Preserve the route-specific existing 404 or redirect behavior in FR-9. |
| Failure | Edit POST has invalid model data or missing/invalid antiforgery token. | Invalid model data remains on Edit without saving; a missing/invalid token is rejected without updating. |
| Boundary | No inactive books exist, while the initialized database still contains other books. | Show a clear no-inactive-books message and a way to return to active books. |
| Boundary | Published year is blank, 1800, 2100, 1799, 2101, or nonnumeric. | Blank, 1800, and 2100 retain existing acceptance; 1799, 2101, and nonnumeric input are rejected by existing validation. |
| Boundary | Title, author, or ISBN reaches or exceeds its current length limit. | Preserve required/string-length rules of 200, 100, and 13 characters; do not add format rules. |
| Boundary | A displayed synthetic title contains HTML markup. | Razor displays it as text, not executable or interpreted markup. |
| Compatibility | Direct Details/Edit/Delete route for an inactive record, Cancel/Back navigation, or successful Delete. | Preserve existing direct-route access and Index navigation. |
| Compatibility | A stale Edit form is submitted after another Edit saved a different title, without a delete or DB failure. | Preserve existing behavior without concurrency control: the later valid save's submitted title remains. Do not promise success for delete/save races. |
| Compatibility | App restarts against the same disposable database. | Previously saved status persists. |

## Acceptance criteria

All data used for checks must be synthetic. Use distinct simple ASCII titles
for ordering assertions; do not assume SQL and in-memory collation match for
all strings. Establish initial state without deleting or modifying unowned
data.

### AC-1: Active default and navigation

- **Given** active and inactive synthetic books in the initialized database
- **When** a user opens `/` or `/Books`
- **Then** only active books appear in ascending title order, and a link to the inactive-only view is available
- **Verification:** Focused automated check for filtering and ordering; manually inspect both routes in the disposable running app.

### AC-2: Inactive discovery and actions

- **Given** one inactive-on-create book, one book deactivated through Edit, and at least one active book
- **When** a user opens the inactive-only view
- **Then** both inactive books appear in ascending title order, active books do not appear, and each row has the existing Details, Edit, and Delete actions
- **Verification:** Automated filtering/order test plus manual IIS Express/EF6 check, including each action link.

### AC-3: Empty inactive state

- **Given** an initialized, nonempty disposable database with no inactive books
- **When** a user opens the inactive-only view
- **Then** a clear no-inactive-books message and a way back to active books appear
- **Verification:** Automated empty-result test and manual view check; keep the table nonempty so startup seeding does not change the setup.

### AC-4: Reactivation

- **Given** an inactive book shown in the inactive-only view
- **When** a user opens its existing Edit form, selects Active, and submits valid data with a valid antiforgery token
- **Then** the book is saved active, the response redirects to the active catalog, and subsequent list requests show it only as active
- **Verification:** Automated controller behavior where deterministic; manually verify the rendered form, EF6 persistence, redirect, and both lists.

### AC-5: Current status

- **Given** a book's status has been changed and saved
- **When** either list is requested again
- **Then** its inclusion reflects the current saved status
- **Verification:** Manual consecutive GETs after create, deactivation, and reactivation.

### AC-6: Existing access and navigation

- **Given** a user who can access the existing catalog and a book of either status
- **When** the user opens the inactive view, a row action, or a direct Details/Edit/Delete GET
- **Then** existing access and direct-route behavior is preserved; Edit Cancel, Details Back, Delete Cancel, and successful Delete POST lead to active Index
- **Verification:** Manual route and navigation checks in the disposable app. Host-level authorization is unknown and is not asserted.

### AC-7: Edit validation

- **Given** an existing book and an Edit submission with a missing title, missing author, or a published year outside 1800-2100
- **When** the user submits it
- **Then** the existing Edit validation response is shown and the stored book is unchanged
- **Verification:** Manual invalid-form checks for each case; inspect persisted synthetic data.

### AC-8: Input boundaries

- **Given** otherwise valid synthetic book fields
- **When** the user submits blank, 1800, 2100, 1799, 2101, and nonnumeric published years, plus title/author/ISBN values at their documented string length boundaries
- **Then** existing rules accept blank year and inclusive endpoints, reject out-of-range/nonnumeric years, and enforce required title/author and 200/100/13 maximum lengths without a new ISBN format rule
- **Verification:** Tests or manual form submissions against the existing validators; record actual outcomes.

### AC-9: Missing records and action methods

- **Given** an ID that does not exist (or a book removed after its form was loaded)
- **When** Details, Edit, or Delete GET, a valid Edit POST, an invalid Edit POST, or a token-valid Delete POST is requested
- **Then** existing behavior remains: missing Details/Edit/Delete GET and valid Edit POST return 404; invalid Edit POST retains validation precedence; missing Delete POST redirects to Index
- **Verification:** Manual requests with a disposable database. Do not infer the result from controller unit tests alone.

### AC-10: Antiforgery

- **Given** an Edit or Delete POST with a missing or invalid antiforgery token
- **When** it is submitted
- **Then** the request is rejected and no update/deletion is performed
- **Verification:** Manual HTTP/form checks against the disposable running app; confirm stored data remains unchanged.

### AC-11: Query failure behavior

- **Given** an isolated, safely induced database query failure
- **When** either catalog list is requested
- **Then** the response does not present failure as a normal empty result and matches the observed baseline error handling
- **Verification:** Compare baseline and changed app only with a verified task-owned isolated mechanism. If unavailable, stop and record this check as unverified; never fault-inject against shared LocalDB.

### AC-12: Save failure behavior

- **Given** an isolated, safely induced Edit save failure
- **When** a valid Edit POST is submitted
- **Then** no successful-save redirect is reported and the observed baseline error handling is preserved; the persisted state is checked separately where safe, without assuming rollback
- **Verification:** Compare baseline and changed app using a verified isolated mechanism. If unavailable, stop and record the check as unverified; no automatic retry.

### AC-13: Restart persistence

- **Given** an edited synthetic book and a verified task-owned disposable database
- **When** the application is stopped and restarted against the same database
- **Then** the saved status remains
- **Verification:** Documented IIS Express/LocalDB persistence check.

### AC-14: Read-only list navigation

- **Given** an initialized, nonempty disposable database with known synthetic records
- **When** a user navigates between active and inactive lists
- **Then** the navigation itself does not change the records
- **Verification:** Compare synthetic records before and after the GET requests.

### AC-15: Encoded display

- **Given** a synthetic title containing HTML markup
- **When** either list displays that title
- **Then** the title is rendered as text and not interpreted as markup or script
- **Verification:** Inspect encoded response/browser rendering; do not use raw-HTML rendering for book fields.

### AC-16: Stale edit compatibility

- **Given** two Edit forms loaded for the same synthetic book
- **When** the first saves title A and the second then saves title B, without deletion or database error
- **Then** title B remains on the next read, preserving the existing no-concurrency-token behavior
- **Verification:** Manual sequential form submissions in the disposable app. A delete/save race is not required to succeed.

## Non-functional requirements

| ID | Quality | Testable requirement | Verification |
| --- | --- | --- | --- |
| NFR-1 | Reliability | A list or save failure is not converted into an empty list or a successful-save response; no automatic retry or rollback claim is added. | AC-11 and AC-12 against a safely isolated failure mechanism; otherwise report unverified and stop. |
| NFR-2 | Security/privacy | New listing uses existing access behavior and Razor text encoding. Only synthetic data is used; secrets and real personal/participant/customer/tenant data are not logged, committed, or exposed. | Review changes and synthetic browser checks; host-level authorization and data classification remain unknown. |
| NFR-3 | Compatibility | Keep .NET Framework 4.8, MVC 5, EF6, existing schema, routes, active-only default, validation, antiforgery, and CRUD behavior. | Primary build plus AC-1, AC-6 through AC-10, AC-13 and AC-16. |
| NFR-4 | Testability | Focused MSTest cases run deterministically without IIS, LocalDB, Azure, or network after restore; tests execute rather than merely discover zero cases. | Record exact test-runner command and executed count. Framework/package/adapter/runner compatibility is currently unknown. |

## Assumptions, dependencies, and risks

| Item | Type | Validation or mitigation |
| --- | --- | --- |
| Existing catalog users may discover inactive records. | Product decision (confirmed) | User explicitly accepted this visibility change; no new roles or authorization behavior are specified. |
| Real deployments may contain sensitive book metadata or add host-level authorization. | Risk / unknown | Not established by repository evidence. Use synthetic data; stop if satisfying the requirement exposes sensitive data or conflicts with actual host policy. |
| A compatible .NET Framework 4.8 MSTest package set and Visual Studio test runner are available. | Dependency (unknown) | Establish versions and invocation before implementation; stop if unavailable or requiring unapproved paths/dependency changes. |
| The available Windows environment has MSBuild/web tooling, .NET Framework 4.8 targeting tools, IIS Express, and SQL Server LocalDB. | Dependency (unknown) | Check readiness; do not report success until observed. |
| The LocalDB instance and files used by manual checks can be made disposable and task-owned. | Safety dependency (unknown) | Verify exact instance/file/process ownership before any writes, restart, fault injection, or cleanup. |
| Safe isolated query/save failure injection can be achieved. | Validation risk (unknown) | Investigate only within a task-owned disposable environment; otherwise record the criterion unverified and stop, without touching shared data. |
| The legacy initializer may drop/recreate on model change and seeds when the table is empty. | Risk supported by evidence | `Models\ApplicationDbContext.cs:20-25`; `Global.asax.cs:14-26`. Preserve schema and nonempty setup for empty-inactive checks; do not use empty MDF workaround. |
| Build/restore may create generated package and build outputs. | Operational risk | Do not hand-edit or commit generated `packages`, `bin`, or `obj` content. |
| Exact app baseline and validation outcomes are not yet observed. | Evidence gap | Build and manual checks must be run and recorded; this specification does not claim a pass. |

## Plan-mode review checkpoint

### Evidence

- **Current behavior:** `legacy-src\dotnet-bookcatalog\shared-legacy-app\src\BookCatalog.Web\Controllers\BooksController.cs:13-19`, `BooksController.Index` — active-only title-ordered query; `Views\Books\Index.cshtml:29-49` exposes actions only for returned rows; `BooksController.cs:65-82` saves Active on Edit and redirects to Index.
- **Existing tests:** No automated baseline test project is documented in `legacy-src\dotnet-bookcatalog\AGENTS.md`; `BookCatalog.sln` currently contains only the web project. The modernized-target smoke at `scripts\validation\Test-BookCatalogSmoke.ps1:163-178` is not a baseline test and does not check UI discovery.
- **Baseline validation evidence:** `msbuild .\shared-legacy-app\BookCatalog.sln -restore -p:Configuration=Release` from `legacy-src\dotnet-bookcatalog` — not run for this change; no exit status or result is claimed. Documented prerequisites include Windows, Visual Studio MSBuild/web tooling, .NET Framework 4.8 targeting tools, IIS Express, and SQL Server LocalDB (`legacy-src\dotnet-bookcatalog\AGENTS.md`).

### Unresolved questions

- [x] **Owner/reviewer:** Unknown; assign before approval.
- [x] **Related issue:** None created.
- [x] **Baseline and environment:** Not run; establish required tools and record actual build and manual outcomes before implementation.
- [x] **Test dependencies and runner:** MSTest selected, but exact package versions, adapter, Visual Studio test runner executable, and command are unknown; verify compatibility before implementation.
- [x] **Failure checks:** A safely isolated query/save failure-injection method is unknown; if unavailable, stop and report these acceptance checks as unverified.
- [x] **Host policy/data classification:** The user approved visibility for existing catalog users, but any actual host-level authorization and sensitivity of real book data are unknown. Use only synthetic data and stop if a real environment would expose sensitive information.

No unresolved product behavior is intentionally left to implementation judgment. The unknowns above are prerequisites/evidence gaps; if they change scope, safety, allowed paths, or acceptance criteria, obtain human guidance and revise this specification before implementation.

### Boundary and validation review

- [x] Evidence citations were checked against the repository.
- [x] Allowed paths enumerate anticipated application and test changes inside the selected .NET source root; the requested spec file is separately identified.
- [x] The primary validation command is the documented Release build from the selected source root; it is not claimed to run behavior tests.
- [x] Acceptance criteria cover normal, failure, boundary, privacy, and compatibility behavior. Failure-injection validation is explicitly conditional on a safe isolated mechanism.
- [x] Exclusions prevent unrelated modernization, schema changes, access-policy changes, and cleanup.
- [ ] Primary build, test runner, and manual environment prerequisites have not yet been verified.

### Stop conditions

Stop and request human guidance if:

- repository evidence contradicts this specification;
- an unresolved question could materially change behavior, access, safety, scope, acceptance criteria, allowed paths, exclusions, or validation;
- satisfying a criterion requires a source path outside the allowed paths;
- a schema change, framework upgrade, new authorization rule, public contract change, dependency/version change beyond the approved focused test setup, or other out-of-scope change is required;
- the pre-change baseline or primary validation fails, or the primary command is unavailable, destructive, or irreproducible;
- Visual Studio/MSBuild/web tooling, .NET Framework targeting tools, IIS Express, LocalDB, test runner, package restore, approval, or another required prerequisite is missing;
- zero automated tests execute, any required test/build/manual check fails or cannot be performed, or safe failure injection cannot be isolated; do not weaken or skip a check or claim an unobserved pass;
- task-owned disposable database/process/file ownership cannot be verified before a destructive action, or a check would affect shared, production, or business data;
- a secret, credential, real personal, participant, customer, or tenant data would be needed, committed, logged, or exposed; or
- implementation/validation cannot meet the acceptance criteria without expanding the approved contract.

Never create an empty MDF to work around initialization, hand-edit generated
files, delete unowned data, or infer rollback from a failed save.

### Human approval

- [x] I reviewed the evidence, unresolved questions, allowed paths, acceptance
      criteria, validation command, exclusions, risks, and stop conditions.
- **Reviewer:** Bing Liu
- **Approval date:** 2026.11.23
- **Approved specification commit:** 364c8d8065960680b087d3195b3df1e61212357f

Keep **Status** as `Draft` until the human approval checkbox is selected and
reviewer, date, and commit fields are complete. Approval of the reviewed plan
does not itself approve this durable specification or authorize implementation.
