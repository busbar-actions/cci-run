> [!WARNING]
> **`busbar-actions` is under heavy active development — expect breaking changes.**
> These repositories are public, but **not ready for use yet** — please don't depend on them.
> A pilot is starting soon: **[star and watch the busbar-actions organization](https://github.com/busbar-actions)** for the launch of Discussions and the pilot announcement.

# busbar-actions/cci-run

Run **one CumulusCI task** against a Busbar-equipped Salesforce org, with the
Salesforce session held outside the task and every request it sends inspected
before the session is attached:

- `task_query` — one Bulk API 1.0 query, written as CSV into your workspace;
- `task_execute_anon` — anonymous Apex **committed in your repository**, run
  only when your repository's own policies permit it.

There are no free-form `cci` arguments, flows or other tasks.

**If the request policy cannot inspect a request, the request gets no
credential.** There is no unenforced mode, opt-out, environment variable or
toggle, on any kind of org.

Three things decide what a run may do, in this order:

1. **Busbar's Baseline**, compiled in and never overridable (no credential
   endpoints, no credential-bearing records, no Busbar trust records, no
   request the policy cannot read in full, …);
2. **the action's grant**: exactly what `task_query` or `task_execute_anon`
   needs and nothing else;
3. **your repository's Cedar policies** (`.busbar/policies/*.cedar` on your
   default branch): they can forbid anything, and anonymous Apex needs one of
   them to permit it. They decide on the run's **run context**: what the run
   declared, what it has done so far, and **risk scores** Busbar computes for
   every query and every piece of Apex.

## How it works

1. **Install.** The action sets up CPython 3.12 (`actions/setup-python`,
   without changing your job's environment) and runs `cci-run install` with a
   **hash-locked requirements file**: this action's own
   [`cci-lock.txt`](cci-lock.txt) (CumulusCI 4.10.1) unless you pass your own
   (`requirements`, see [Choosing the CumulusCI version](#choosing-the-cumulusci-version)).
   For each locked version one file is chosen through PyPI's JSON API
   (`pypi.org`) among the files whose SHA-256 the lock lists, downloaded over
   HTTPS from `files.pythonhosted.org` only, checked against that SHA-256 and
   a size cap, and installed with `pip --require-hashes --no-index` **inside
   the sandbox with the network blocked** and an empty environment. Nothing
   unhashed is ever installed. The Busbar CumulusCI keychain is embedded in
   the binary.
2. **Plan.** The action document is checked before any session exists: the
   SOQL is parsed (only what Bulk API 1.0 supports; any syntax error
   refuses), `FROM` is the declared object, no credential-bearing object or
   field names, no `$project_config` substitutions; `apex_file` must be a
   plain repository-relative path, and inline `apex` is refused. Your
   `cumulusci.yml` must be a regular file (not a symbolic link), must not
   override the `query` or `execute_anon` task, and its
   `project.package.api_version`, if set, must be written exactly as
   CumulusCI sends it (`"60.0"`: two or three digits, `.0`).
3. **CumulusCI reads your project file.** Before anything is minted, the
   installed CumulusCI's own YAML loader reads the copied `cumulusci.yml` in
   a confined child with no network and no credential. The same rules are
   applied to what *it* read, so YAML that reads differently to different
   parsers (an explicit `!!merge` tag, for example) cannot hide a task
   override. It also reports the CumulusCI version (which must be the one
   your lock pins) and, when your project sets no `api_version`, the
   CumulusCI default API version the run will use.
4. **The decode helper.** cci-run starts a copy of itself as a **confined,
   credential-free helper**: no network (on Linux no socket at all), an
   empty environment, nothing readable but its own executable, started
   before any credential exists. Every piece of content that a run could
   make hostile — the SOQL, the Bulk job documents CumulusCI sends, the
   committed Apex — is parsed only there. cci-run checks every answer
   against the bytes that were sent and makes every decision itself; a
   crash, a timeout or an answer that does not check out is a denial.
5. **Mint and bind.** The binary mints its own short-lived Salesforce
   session from the runner's GitHub OIDC token and holds it in its own
   memory. Then, before CumulusCI starts:
   - it reads your repository's **policies**: the default branch's head
     commit, whether GitHub reports the branch protected (recorded, not
     enforced), and every file in `.busbar/policies/` at that commit, through
     the GitHub REST API with `github-token`. An invalid policy set fails the
     run here;
   - it reads your repository's **door configuration**,
     `.busbar/door.toml`, at the same commit, if there is one (see
     [Header forwarding](#header-forwarding-busbardoortoml)). An invalid one
     fails the run here;
   - for `task_execute_anon`, it reads **`apex_file` at the commit your
     job's signed OIDC token names** (`sha`), not from the checkout, and the
     helper analyzes it;
   - for `task_query`, it reads the org's describes for every object the
     SOQL reaches (parent paths, polymorphic targets, `TYPEOF`, semi-joins);
     any name it cannot resolve, or that resolves to a credential-bearing
     object or field, refuses the run;
   - it writes the run's **declaration** (see [The run context](#the-run-context-and-risk-scores)),
     scores the query or the Apex, and asks your policies about every
     operation the run declares. Anonymous Apex that no policy permits, or
     any operation a policy forbids, stops here.
6. **Run.** CumulusCI runs from a private shadow project (only
   `cumulusci.yml` is copied; `.git` is never read) with a private HOME, an
   explicit environment, read-only access to the Python install and TCP only
   to the in-process proxy. The proxy shows the policy every request **in
   full** — request line, headers, body — before injecting the session. It
   decides by what the request does, whichever CumulusCI release sent it:
   the Bulk job documents must say what your action declared (a CSV query
   of your object), the batch must be your SOQL byte for byte at the run's
   one API version, follow-up requests must name the job, batch and result
   ids Salesforce returned for this run (as many status polls and result
   lists as the task's time limit allows at two a second, paced from the
   batch: 64 at once, then two a second), and the committed
   Apex is sent exactly once, byte for byte. Before the policy sees a
   request, the proxy removes every header whose name it does not forward,
   and the request goes on without it; the headers it forwards are checked
   by meaning (see [Header forwarding](#header-forwarding-busbardoortoml)). Your policies are asked again about every request,
   with everything the run has done so far. A denied request reaches
   CumulusCI as HTTP 403 without the credential. The task has a time limit
   (60 minutes for `task_query`, 15 for `task_execute_anon`); past it the
   session is shut down and the step fails.
7. **Finish.** The report (with the whole run context) is written to
   `policy-out`, the result file is copied into `working-directory`, the
   session is revoked and zeroized.

The job must:

1. grant `permissions: id-token: write, contents: read` — every run reads
   your repository's policies, and `task_execute_anon` its Apex file, with
   `github-token` (by default the job's own token);
2. set `target-instance` to the org's instance URL (the org must trust this
   repository, workflow and environment: see
   [busbar-actions/sf-busbar-setup](https://github.com/busbar-actions/sf-busbar-setup),
   which checks that the environment and your default branch are protected
   when it creates that trust);
3. run on `ubuntu-latest` (Linux x86_64 with Landlock ABI v4, kernel 6.7+;
   session set-up fails closed without it).

## The action document

```json
{ "action": "task_query", "object": "Account", "soql": "SELECT Id, Name FROM Account WHERE CreatedDate = LAST_N_DAYS:7", "result_file": ".busbar/out/accounts.csv" }
```
```json
{ "action": "task_execute_anon", "apex_file": "scripts/apex/seed-accounts.apex" }
```

- **`task_query`**
  - `object` is sent in the Bulk job, so spell it as the `FROM` object is
    spelled.
  - `soql` is the batch body, byte for byte. Accepted: fields, parent paths,
    polymorphic paths, `TYPEOF`, functions (`toLabel`, `convertCurrency`,
    `FORMAT`, date functions, `DISTANCE`/`GEOLOCATION`), `FIELDS(STANDARD |
    CUSTOM | ALL)` (expanded through the org's describe, so every field it
    reaches is checked), `WHERE` (including semi-joins and anti-joins),
    `WITH SECURITY_ENFORCED | USER_MODE | DATA CATEGORY`, `USING SCOPE`,
    `ORDER BY`, `LIMIT`. Refused, because Bulk API 1.0 does not support them
    or they cannot be resolved exactly: child subqueries in the select list,
    aggregates and `COUNT()`, `GROUP BY`/`ROLLUP`/`CUBE`, `HAVING`,
    `OFFSET`, `FOR VIEW | REFERENCE | UPDATE`, `WITH SYSTEM_MODE`, `ALL
    ROWS`, bind variables, comments, a field or object named like a SOQL
    keyword, and anything over 20,000 bytes. `AND` and `OR` do not mix
    without parentheses, and `NOT` next to either needs them too: write
    `A OR (NOT B)`. Every field any clause names (`WHERE`, `ORDER BY`,
    function arguments, …) is resolved and checked.
  - `result_file` is relative to `working-directory`: no `..`, not absolute,
    not under `.git`. The run writes a private file; it is copied into place
    only after a successful run.
- **`task_execute_anon`**
  - `apex_file` is a path **relative to the repository root** (not to
    `working-directory`): no `..`, not absolute, not under `.git`, and no
    symbolic link or submodule anywhere on it. The file is read at the
    commit your job's signed OIDC token names — the commit the workflow run
    is for — so it must be **committed and pushed**; editing it in an
    earlier step changes nothing.
  - What is sent is `"\n"` followed by the file's bytes, unchanged (CRLF, a
    BOM and tabs included). The file must be non-empty UTF-8 with no NUL and
    no control character other than TAB, LF or CR, no `%%%` token and no
    `$project_config` reference (CumulusCI would rewrite those, so the bytes
    sent would not be the bytes committed), not the sandbox's credential
    placeholder, and fit 16,323 bytes form-encoded (CumulusCI sends it in
    the request line, and Salesforce's REST URI limit is 16,384 bytes).
    Quotes, parentheses and semicolons take three bytes each form-encoded,
    so this is roughly 6 to 10 KB of ordinary Apex.
  - Its digest is `sha256("\n" + file)` (hex): the bytes Salesforce compiles
    (the `apex-digest` output).
- Any other field — including the old `org` and the inline `apex` — is
  refused.

## Anonymous Apex: committed, analyzed, and permitted by your policies

Anonymous Apex is an **approval-class** operation: it runs only when a policy
in your repository's `.busbar/policies/*.cedar` **permits** it and **no
policy forbids** it — on every org kind, scratch orgs included. With no
policy files, it never runs. There is no `approved-digests` input any more
and no check of your GitHub Environment's protection, at run time or at
setup: GitHub enforces Environment protection only on some plans, so your
policies (and whatever protection GitHub enforces for you) are the gate.

Before your policies are asked, the Apex is analyzed (in the credential-free
helper) and scored. Your policy sees, under `context.apex`: the file's
`path`, the `commit` it was read at, its `digest`, the `api_version` it runs
at (at the door, the version cci sends `executeAnonymous` at; at bind, the
declared one) and the `declared_api_version` (the one the installed
CumulusCI sends at), its `risk` score, whether it is `destructive` (it can
update, upsert, delete or merge records, does DML on an object the code does
not name, calls code already deployed in the org, or could not be analyzed),
and the analyzer's `facts`: `dml_ops` and `dml_objects`, `soql_objects`,
`dynamic_dml`, `dynamic_soql`, `callouts`, `without_sharing`,
`dynamic_dispatch` (`Type.forName` and similar), `session_access`
(`UserInfo.getSessionId` and similar), `run_as`, `async_jobs`, `email`,
`platform_events`, and `calls_org_code` with the `org_types` it reaches
(classes already deployed in the org, whose behaviour the analyzer cannot
see; a standard object named only as a type counts too). Every set of
object names your policies see (`context.apex.facts.dml_objects`,
`soql_objects`, `org_types`, `context.query.objects`,
`context.run.objects_touched`) is **lower-case** (`"account"`), however the
object was reached; a policy that compares one with a name holding an
upper-case letter could never match, and does not load.

The door does not pin the API version the Apex runs at. To pin it to the
version the installed CumulusCI declared:

```cedar
@id("apex-at-its-declared-version")
forbid (principal, action == Busbar::Action::"apex.execute", resource)
when { context.apex.api_version != context.apex.declared_api_version };
```

Permitted Apex runs with the run-as user's full power: the permit is the
control. Write narrow permits.

## Writing repository policies

Policies live in **`.busbar/policies/*.cedar`** on your repository's
**default branch**. Every run reads them from the head of the default branch
as it is when the run starts — not from the branch or commit the run is for —
so a policy change takes effect once it is merged. Protect the default branch
(sf-busbar-setup requires it when it creates the trust rule) so a policy
change needs review. The report records which commit the policies were read
at and whether GitHub reported the branch protected.

They are [Cedar](https://www.cedarpolicy.com/) policies, validated **strictly**
against the Busbar Cedar schema (version `busbar-cedar/1`, published as
[`busbar.cedarschema`](https://github.com/composable-delivery/busbar-extensions/blob/main/docs/schemas/busbar.cedarschema)).
The set is all or nothing: a syntax error, a template (`?principal`), a
validation error **or warning** (for example a condition that can never be
true, or mixed-script text), a `.cedar` file in a subdirectory, a hidden one
or one with an upper-case extension, non-UTF-8 text, more than 64 files, a
file over 64 KiB, 512 KiB in all or more than 1,024 policies **fails the
run** before anything starts, with the file, line and column. Other files in
the directory (a `README.md`) are ignored and listed in the report.

### What a policy can see

| Cedar | What it is |
|---|---|
| `principal` (`Busbar::Job`) | The job, from its signed OIDC claims: `repository` (`owner/name`), `repository_owner`, `workflow_ref` (`owner/name/.github/workflows/x.yml@refs/heads/main`), `environment` (`""` when none), `git_ref`, `commit_sha`, `event_name`, `actor`, `run_id`, `issuer`. |
| `action` (`Busbar::Action::"…"`) | The operation: `"query"`, `"query.page"`, `"bulk.job.create"`, `"bulk.batch.add"`, `"bulk.job.close"`, `"bulk.read"`, `"apex.execute"`, `"identity"`, `"api.discovery"`, `"record.read"`, `"record.write"`, `"code.invoke"`, `"apex.run_tests"`, `"other"`. `Busbar::Action::"approval"` groups the approval-class ones (`"apex.execute"` today). |
| `resource` (`Busbar::Org`) | The target org: `org_kind` (`"scratch"`, `"sandbox"` or `"production"` — anything doubtful is `"production"`), `org_id` (18 characters), `instance_host`. |
| `context.phase` | `"bind"` (asked once before the task starts, for every operation the run declared) or `"request"` (asked again for every request). |
| `context.action` | The Busbar action, e.g. `"cci-run:task_execute_anon"`. |
| `context.request` | `route`, `method`, `api_version` of the request (`""` at bind). |
| `context.query` | The query this operation runs: `present`, `declared`, `parsed`, `api` (`"data"`/`"tooling"`), `risk` (0–100), `objects` (resolved API names), `fields` (count), `sensitive_classes`, `access` (`"rows"`, `"grouped"`, `"aggregate"`), `has_where`, `has_limit`, `limit` (`-1` when none), `for_update`, `for_view`. |
| `context.apex` | The Apex this operation runs (see above). |
| `context.side_effect` | This operation's side effect: `present`, `kind` (`"bulk_job_created"`, `"bulk_batch_added"`, `"bulk_job_closed"`, `"code_executed"`, `"row_locks"`, `"view_stamps"`, `"record_write"`), `destructive`. |
| `context.run` | The run **so far** (before this operation): `max_risk`, `cumulative_risk`, `requests_allowed`, `requests_denied`, `queries`, `unparsed_reads`, `side_effects`, `destructive_side_effects`, `objects_touched` (a set, at most 256 names), `objects_touched_truncated`, `sensitive_classes`. |

Every attribute is always present: a part that does not apply says
`present: false` and carries zeros, so a condition never fails on a missing
attribute. Nothing in the context is a token, a record value, or query or
Apex text: only resolved names, digests, counts, flags and scores.

### How policies combine

- `forbid` wins over `permit`, and a `forbid` can stop **any** operation.
- `permit` matters only for approval-class operations (`"apex.execute"`):
  they need one. Every other operation runs unless forbidden.
- A policy can never allow what Busbar's Baseline or the action's grant
  denies: it is asked only about what those two allowed, and the schema has
  no action for anything the Baseline denies.
- A policy that fails to evaluate (for example an arithmetic overflow)
  denies.

### Examples

```cedar
// Anonymous Apex may run in scratch orgs when Busbar scores it below 30:
// never a callout, an email, a platform event, dynamic dispatch, a call into
// deployed org code or an async job (each scores 30 or more), but DML on
// objects the analyzer names may (an insert scores 10). "Below 30" is not
// "read-only"; that is `!context.apex.destructive`.
@id("low-risk-apex-in-scratch")
permit (principal, action == Busbar::Action::"apex.execute", resource)
when { resource.org_kind == "scratch" && context.apex.risk < 30 };
```

```cedar
// No query with a risk score of 70 or more against production: the
// query itself, its next pages, and a Bulk batch carrying it. Every query
// that returns personal data or secrets for more than 200 records scores
// 70 or more, whatever its WHERE clause says.
@id("no-high-risk-soql-in-production")
forbid (principal, action, resource)
when {
  resource.org_kind == "production" &&
  context.query.present && context.query.risk >= 70
};
```

```cedar
// At most one destructive side effect per run.
@id("one-destructive-side-effect-per-run")
forbid (principal, action, resource)
when {
  context.side_effect.destructive &&
  context.run.destructive_side_effects >= 1
};
```

```cedar
// The seed script may run only in the `uat` environment ...
@id("seed-apex-only-in-uat")
permit (principal, action == Busbar::Action::"apex.execute", resource)
when {
  context.apex.path == "scripts/apex/seed.apex" &&
  principal.environment == "uat"
};

// ... and never anywhere else, even where a broader permit would allow it.
@id("seed-apex-never-outside-uat")
forbid (principal, action == Busbar::Action::"apex.execute", resource)
when { context.apex.path == "scripts/apex/seed.apex" }
unless { principal.environment == "uat" };
```

More patterns:

```cedar
// Apex may run in sandboxes only from the release workflow on main, and
// only if it reads and writes no user or session data.
@id("release-apex-in-sandboxes")
permit (principal, action == Busbar::Action::"apex.execute", resource)
when {
  resource.org_kind == "sandbox" &&
  principal.workflow_ref like "*/.github/workflows/release.yml@refs/heads/main" &&
  !context.apex.facts.session_access &&
  !context.apex.facts.run_as &&
  !context.apex.facts.dml_objects.contains("user")
};

// No Apex that can send data out of the org or read the session,
// anywhere, whatever permits it.
@id("no-apex-egress-or-session-access")
forbid (principal, action == Busbar::Action::"apex.execute", resource)
when {
  context.apex.facts.callouts || context.apex.facts.email ||
  context.apex.facts.platform_events || context.apex.facts.session_access
};

// In production, no query that returns personal data or secrets for more
// than 200 records. Only a LIMIT bounds a read: a WHERE clause
// (`WHERE CreatedDate > 2000-01-01T00:00:00Z`) bounds nothing a policy can
// see, so a rule keyed on `context.query.has_where` is evaded by any filter.
@id("no-bulk-personal-data-in-production")
forbid (principal, action, resource)
when {
  resource.org_kind == "production" && context.query.present &&
  context.query.access != "aggregate" &&
  (!context.query.has_limit || context.query.limit > 200) &&
  context.query.sensitive_classes.containsAny([
    "contact_info", "personal_identity", "financial", "health",
    "encrypted", "credential_adjacent"
  ])
};

// Stop a run once its accumulated risk is high.
@id("cumulative-risk-budget")
forbid (principal, action, resource)
when { context.run.cumulative_risk >= 250 };
```

A `forbid` that names an object in `context.run.objects_touched` should also
forbid when `context.run.objects_touched_truncated` is true (more than 256
objects were touched, so the set is cut):

```cedar
@id("never-touch-opportunity")
forbid (principal, action, resource)
when {
  context.run.objects_touched.contains("opportunity") ||
  context.run.objects_touched_truncated
};
```

When a run is refused because no policy permits its Apex (selector
`approval-class`), the refusal names the operation and shows an example
permit to start from.

## Header forwarding: `.busbar/door.toml`

Before the request policy sees a request, the proxy removes every header
whose name it does not **forward**, and the request goes on without it. The
policy is shown exactly the request that is forwarded, so a header a new
CumulusCI (or Python library) release adds never breaks a run, and never
reaches your org. The proxy always forwards:

- the headers the policy checks by meaning: `Accept`, `Accept-Encoding`,
  `Accept-Language`, `Accept-Charset`, `X-PrettyPrint`, `Content-Type`
  (checked per route), `User-Agent`, `Sec-Fetch-Mode`, the credential slots
  it fills itself (`Authorization`, `X-SFDC-Session`: CumulusCI only ever
  holds a placeholder, and its own value is never forwarded),
  `Sforce-Call-Options` (`client=` only) and `Sforce-Query-Options`
  (`batchSize=` 200–2000, on a query route);
- the headers whose presence refuses a request (`B5`), so they are refused
  as before rather than silently dropped: `X-HTTP-Method-Override`,
  `X-HTTP-Method`, `X-Method-Override`, `Content-Encoding` (removing it
  would forward an encoded body the policy had read as plain), `Keep-Alive`,
  `TE`, `Trailer`, `Proxy-Authenticate` and `HTTP2-Settings`.

Everything else is removed: every other `Sforce-*` option (PK chunking,
assignment or duplicate rules, …), cookies, conditional and range headers,
tracing headers, anything unknown. A request line or header the proxy and
Salesforce could read differently (a folded line, whitespace before a
colon, a control character, conflicting lengths) is still refused before
anything is removed.

If your runs need another header forwarded, list it in **`.busbar/door.toml`**
on your **default branch** (read at the same commit as `.busbar/policies`):

```toml
# .busbar/door.toml
forward_headers = ["traceparent", "X-Request-Id"]
```

- `forward_headers` is the only key: at most 64 names, each a valid HTTP
  header name of at most 128 bytes, each listed once (neither case nor `_`
  for `-` makes a name another one). A name the proxy already forwards,
  spelled as it is, is accepted and changes nothing.
- Never forwardable, whatever the file says (compiled in): credential
  headers (`Authorization`, `Proxy-Authorization`, `X-SFDC-Session`,
  `Cookie`, `Cookie2`), framing and hop-by-hop headers (`Host`,
  `Content-Length`, `Transfer-Encoding`, `Connection`, `Keep-Alive`, `TE`,
  `Trailer`, `Upgrade`, `Expect`, `Content-Encoding`, `HTTP2-Settings`, …),
  method, host and target overrides (`X-HTTP-Method-Override`,
  `X-HTTP-Method`, `X-Method-Override`, `X-Original-Method`,
  `X-Original-URL`, `X-Rewrite-URL`, `X-Override-URL`,
  `X-HTTP-DestinationURL`, `X-Host`, `X-HTTP-Host-Override`,
  `X-Original-Host`) and proxy and client-IP headers (`Proxy-*`,
  `Forwarded`, `Via`, `X-Forwarded` and `X-Forwarded-*`, `X-Real-IP`,
  `True-Client-IP`, `X-Client-IP`, `X-Cluster-Client-IP`, `Client-IP`,
  `CF-Connecting-IP`, `Fastly-Client-IP`, `X-Originating-IP`,
  `X-Remote-IP`, `X-Remote-Addr`). Names are compared with `_` read as
  `-`, as some servers read them: `X_HTTP_Method_Override` is refused like
  `X-HTTP-Method-Override`, and another spelling of a header the proxy
  already forwards and checks (`Content_Type`) is refused too.
- A listed header is forwarded with any value that is valid header syntax,
  except a `Sforce-*` (or `Sforce_*`) header on a Bulk API 1.0 request (it
  would change what the job does). Your Cedar policies still decide every
  request.
- A file over 64 KiB, not UTF-8, that does not parse, with another key or
  listing a refused name **fails every run** before anything starts
  (selector `door-config`), with the line and column or the name and why.
  No file means the proxy forwards only the headers above.

The run's declaration records the file (`declaration.door`: repository,
branch, commit, path, blob id, SHA-256, your extra names and every name the
proxy forwards), and the report counts what the proxy removed, by name
(each cut to 64 bytes), never a value (`stripped_headers`).

## The run context and risk scores

Every run keeps an append-only **run context**, and every run's report
carries it (redacted), so each run leaves an auditable record of what it
intended and what it did:

- **The declaration**, written before CumulusCI starts and never changed:
  the action and its typed inputs (the SOQL by SHA-256 digest, its resolved
  objects, field count, sensitive field classes, shape and risk; the Apex by
  path, commit, file and sent digests, the analyzer's facts and its risk),
  the grant, the job's identity from its signed claims, the org, where the
  policies came from (repository, branch, head commit, whether it was
  protected, the set's digest, the schema version) and the scorer versions.
  cci-run adds its own declaration (`cci_declaration`: the redacted inputs,
  the project's and CumulusCI's API version, the CumulusCI version, the
  digests of the requirements lock and the keychain, the Python version).
- **The journal**: every request the proxy decided (the operation, the
  route, the verdict, the rule, the policies that decided it, the query's
  risk, the objects it reached), every side effect (a Bulk job created, a
  batch added, a job closed, Apex executed, rows locked, each marked
  destructive or not), and every fact the run learned (the org's
  classification, the describes read, the Apex analysis, the Bulk job and
  batch ids and result counts Salesforce returned for this run, every
  policy evaluation at bind).
- **Derived facts**, kept current as the journal grows: the highest and the
  cumulative risk, requests allowed and denied, queries, side effects,
  destructive side effects, the objects touched and the sensitive field
  classes read. These are `context.run` in your policies.

### Risk scores

Risk scores come from scorers compiled into the cci-run binary, each
versioned (`busbar/soql-risk/v3`, `busbar/apex-risk/v3`, with the field
table `busbar/sensitive-fields/v1`). A score is 0–100 and carries its
reasons with their points; a change to any weight is a new scorer version,
so a policy threshold never silently changes meaning. The declared
operation's score is the `risk` output, and the job summary lists its
reasons.

**A query** (`busbar/soql-risk/v3`):

| Reason | Points |
|---|---|
| `row-access` (returns records) | 10 |
| `grouped-access` (`GROUP BY`, no sensitive field) | 5 |
| `grouped-sensitive-access` (`GROUP BY` with a sensitive field: its values come back, one per group) | 10 |
| `no-filter` (records or groups, no `WHERE`) | 10 |
| `no-limit` (records or groups, no `LIMIT`) | 10 |
| `large-limit` (`LIMIT` over 2000) | 5 |
| `unnamed-fields` (`FIELDS(ALL)` or `FIELDS(CUSTOM)`) | 10 |
| `contact-info` (email, phone, address, location fields) | 15 |
| `personal-identity` (birth date, national or tax id, passport, …) | 20 |
| `financial` (bank account, card, routing number, income) | 20 |
| `health` (diagnosis, medication, medical, disability) | 20 |
| `encrypted` (an encrypted field) | 25 |
| `credential-adjacent-field` (named like a password, token, secret, key, session, …) | 25 |
| `bulk-sensitive-read` (records or groups with a sensitive field, not bounded to 200 records: no `LIMIT`, a `LIMIT` over 200, or a child subquery) | 45 |
| `credential-adjacent-object` (users, their logins, sessions, permissions, auth configuration) | 20 |
| `many-objects` (per object beyond the first) | 5, at most 15 |
| `for-update` / `view-stamps` | 15 / 5 |

`no-filter` and `no-limit` do not apply to single-row objects such as
`Organization`. A sensitive class counts once per query, wherever the query
names it (a filter on a field discloses its values too). Every query that
returns records or groups, names a sensitive field and is not bounded to
200 records scores at least 10 + 15 + 45 = **70**, whatever its `WHERE`
clause says: only a `LIMIT` bounds a read; a targeted read
(`… WHERE Name = 'x' LIMIT 1`) scores by its classes alone. `WITH
SECURITY_ENFORCED` and `WITH USER_MODE` earn nothing: a query sent through
the API already runs as the API user with object, field and sharing checks
applied. (`busbar/soql-risk/v2` had no `bulk-sensitive-read`: a filtered
read of 5000 contacts' email, phone and birth date scored 50.
`busbar/soql-risk/v1` subtracted 10 for the `WITH` clauses and did not count
`no-limit` on grouped queries.)

**Anonymous Apex** (`busbar/apex-risk/v3`):

| Reason | Points |
|---|---|
| `unanalyzed` (the analyzer could not analyze it) | 100 |
| `dml-insert`, `dml-undelete` | 10 each |
| `dml-update`, `dml-upsert` | 20 each |
| `dml-delete`, `dml-merge`, `dynamic-dml` | 30 each |
| `static-soql` / `dynamic-soql` | 5 / 20 |
| `callouts`, `email` (data can leave the org) | 40 each |
| `platform-events` (subscribers outside the org may receive them) | 30 |
| `dynamic-dispatch`, `async-jobs` (code the analyzer cannot see, or that runs after the run) | 30 each |
| `without-sharing` | 20 |
| `session-access` | 40 |
| `run-as` | 15 |
| `credential-adjacent-object` | 20 |
| `org-code` (calls code already deployed in the org: `MyOrgUtil.run();`) | 30 |

Scores are clamped to 0–100. `System.debug('hello');` scores 0; an `insert`
of Accounts 10; a unit whose only effect is an HTTP callout 40; code that
deletes records and makes a callout at least 70. Every way the Apex can
send data out of the org and every piece of code it runs that the analysis
cannot see scores 30 or more on its own, so a `risk < 30` permit never
admits one; DML and SOQL on objects the analyzer names can stay below 30,
so a risk threshold is not "read-only" (write `!context.apex.destructive`
for that). (`busbar/apex-risk/v2` scored `callouts` 25, `email` 20,
`platform-events` 15, `dynamic-dispatch` and `async-jobs` 20.)

## Choosing the CumulusCI version

Nothing in cci-run is tied to a CumulusCI release: every request is decided
by what it does. The version is whatever your **requirements lock** pins.
This action ships [`cci-lock.txt`](cci-lock.txt) (CumulusCI 4.10.1 and its
whole runtime closure for CPython 3.12 on Linux x86_64) as the default. To
run another release, commit your own lock in the same format and pass its
path:

```yaml
      - uses: busbar-actions/cci-run@v1
        with:
          target-instance: ${{ vars.SF_INSTANCE_URL }}
          requirements: .busbar/cci-lock.txt
          action: |
            {"action":"task_query","object":"Account",
             "soql":"SELECT Id FROM Account LIMIT 10","result_file":".busbar/out/a.csv"}
```

The format is pip's hash-checking mode, one `name==version` per requirement
with `--hash=sha256:<hex>` for every file that may be installed:

```sh
uv pip compile requirements.in --generate-hashes \
  --python-version 3.12 --python-platform x86_64-manylinux_2_28 \
  --no-header -o .busbar/cci-lock.txt
```

cci-run refuses anything looser: pip options (`--index-url`, `-r`, `-e`,
…), direct references, environment markers (as `--universal` locks write
them), ranges, other hash algorithms. Packages come from PyPI only. The lock
must include the build backend its source-only distributions need
(`setuptools`, `wheel`). The CumulusCI version the lock pins must be the one
the installed CumulusCI reports (checked before anything is minted).

When your `cumulusci.yml` sets no `api_version`, the run uses the installed
CumulusCI's own default (67.0 for 4.10.1, 63.0 for 4.8.1).

## Inputs

| Input | Default | Description |
|---|---|---|
| `action` (required) | — | The action document (see above). |
| `target-instance` (required) | — | Instance URL of the Busbar-equipped org. |
| `token-handler` | `` | Apex token-exchange handler dev name. Empty ⇒ `GitHubTokenExchangeHandler`. |
| `eca-client-id` | `` | External Client App consumer key (non-secret). Empty ⇒ busbar-auth's default. |
| `oidc-audience` | `` | OIDC audience. Empty ⇒ `target-instance`. |
| `requirements` | `` | Path of a hash-locked requirements file pinning CumulusCI. Empty ⇒ this action's `cci-lock.txt` (CumulusCI 4.10.1). |
| `github-token` | `${{ github.token }}` | Token cci-run's supervisor uses to read your default branch, `.busbar/policies`, `.busbar/door.toml` and the committed Apex file; needs `contents: read`. Never given to CumulusCI. |
| `working-directory` | `.` | Directory holding `cumulusci.yml`; `result_file` is relative to it. |
| `expect-denial` | `` | Negative test selector (see below). |
| `policy-out` | `<runner.temp>/cci-run-policy.json` | Where the run report is written. |
| `audit-out` | `<runner.temp>/busbar-sandbox-audit.json` | Observed-endpoint declaration (audit only; `[]` when nothing ran). |
| `artifact-name` | `busbar-sandbox-cci` | Uploaded artifact name; unique per cci-run step in a run. |
| `version` | `latest` | `cci-run` release tag in `binary-repo`. |
| `binary-repo` | `busbar-actions/actions-dist` | Where to fetch the binary. |

## Outputs

| Output | Description |
|---|---|
| `success` | `"true"` only when the cci task ran and exited 0. |
| `exit-code` | The cci task's exit code (empty when it never started). |
| `result-file` | `task_query`: the result file, relative to `working-directory`. |
| `apex-digest` | `task_execute_anon`: `sha256("\n" + file)`, hex, of the committed file read at the signed commit. |
| `risk` | The declared query's or Apex's risk score, 0–100 (empty when the run stopped before the session was bound). |
| `cumulusci-version` | The CumulusCI version the installed CumulusCI reports. |
| `org-kind` | `scratch`, `sandbox` or `production` (`unclassified` before the org was read). Anything doubtful is `production`. |
| `claim-status` | `eligible`, `ineligible`, or `refused` when nothing ran. |
| `denials` | How many requests the policy denied. |
| `policy-report` | Path of the run report. |

## Negative tests: `expect-denial`

With a selector, the step passes (exit 0) **only** when the run was refused
with that selector before anything ran, or at least one request was denied
under it, no request was allowed on the denied route, and the cci task failed
or never started. Anything else fails the step and says what happened instead.
A bare `true` is refused.

| Selector | Matches |
|---|---|
| `approval-class` | Anonymous Apex no repository policy permits (including a repository with no policies). |
| `policy` | A repository policy forbade the operation, or failed to evaluate. |
| `B1` … `B11` | A Baseline rule: `B1` undecodable or unknown request (including a query string on a Bulk route, or a Bulk API version other than the run's), `B1a` a request the door did not show in full, `B2` credential-issuing endpoints, `B2r` credential-bearing records, `B3` the Busbar trust root, `B4` code other than the committed Apex (or run twice), `B5` ambiguous, hop-by-hop or framing headers, a method override, a `Content-Encoding`, or a forwarded header whose value its class does not allow (a header the proxy does not forward is removed, not refused), `B6` canonicalization ambiguity (a control character in the Apex file, for example), `B7` protocol upgrades, `B8` size limits (an Apex file too long to send), `B11` the credential placeholder. |
| `grant`, `ledger` | An effect outside the grant; a follow-up the ledger does not admit. |
| a route class | e.g. `tooling_execute_anonymous`, `bulk_v1`, `bulk_v1_status`, `query`. |

```yaml
      - uses: busbar-actions/cci-run@v1
        with:
          target-instance: ${{ vars.SF_INSTANCE_URL }}
          expect-denial: B2r
          action: |
            {"action":"task_query","object":"GithubAppSecret__mdt",
             "soql":"SELECT Id FROM GithubAppSecret__mdt","result_file":"out.csv"}
```

## The run report (`policy-out`)

JSON, redacted (no SOQL, Apex, record values, tokens or header values):

- `{"outcome":"ran", …}` — every key of the request policy's report at the
  top level (`org`, `grant_digest`, `claim_status`, `counts`, `ledger`,
  `bulk_query` with its `api_version`, `decisions` — each with its
  `operation`, `rule`, `verdict`, `"inspection": "full"` on every allow, and
  `policy_reasons` when a policy decided it — and **`run_context`**:
  `declaration`, `journal`, `journal_appended`, `journal_dropped`,
  `derived`), `door` (the door configuration: where it was read and the
  header names forwarded) and `stripped_headers` (the header names the
  proxy removed, with counts, never values), plus `policies` (the policy
  directory's git tree id, the source — repository, default branch, head
  commit, `protected` — the set's digest, each file's SHA-256, each
  policy's `@id` and effect), `apex_file`
  (repository, path, commit, blob id, the SHA-256 of the file and of the
  bytes sent), `describes`, `identity_describes`, `cci_declaration`,
  `child_exit` and `child_timed_out`.
- `{"outcome":"refused","stage":"plan"|"pre_mint"|"bind","selector":…,"detail":…}`
  — nothing ran; `cci_declaration` once it was made, and `trail`: what was
  read before the refusal (the policy set, the door configuration, the Apex
  file record).

For example, `jq '.run_context.declaration.inputs.apex.risk'` shows the
Apex's score and reasons, `jq '.run_context.derived'` the run's totals.

`claim_status` is `ineligible` on every run today. The report names why: the
org's trust rule does not yet pin the executing workflow, and a sandboxed
process can reach the proxy's port on any address (GHSA-6hww), so
non-credential data could leave the sandbox. The credential itself never
reaches CumulusCI.

## Example: a read-only query

```yaml
permissions:
  contents: read
  id-token: write

jobs:
  query:
    runs-on: ubuntu-latest
    environment: busbar-test
    steps:
      - uses: actions/checkout@v6
        with:
          persist-credentials: false
      - id: q
        uses: busbar-actions/cci-run@v1
        with:
          target-instance: ${{ vars.SF_INSTANCE_URL }}
          action: |
            {"action":"task_query","object":"Organization",
             "soql":"SELECT Id, Name FROM Organization",
             "result_file":".busbar/out/org.csv"}
      - run: |
          echo "risk: ${{ steps.q.outputs.risk }}"
          cat ".busbar/out/org.csv"
```

The working directory must contain a `cumulusci.yml`. No policy file is
needed for a query: without policies, only anonymous Apex is refused.

## Example: committed Apex in a scratch org

`scripts/apex/seed-accounts.apex`, committed:

```apex
List<Account> accounts = new List<Account>();
for (Integer i = 0; i < 10; i++) {
    accounts.add(new Account(Name = 'Busbar seed ' + i));
}
insert accounts;
```

`.busbar/policies/apex.cedar`, merged to the default branch:

```cedar
@id("seed-accounts-in-scratch")
permit (principal, action == Busbar::Action::"apex.execute", resource)
when {
  resource.org_kind == "scratch" &&
  context.apex.path == "scripts/apex/seed-accounts.apex" &&
  context.apex.risk < 30 &&
  !context.apex.destructive
};
```

The workflow:

```yaml
jobs:
  seed:
    runs-on: ubuntu-latest
    environment: scratch
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@v6
        with:
          persist-credentials: false
      - uses: busbar-actions/cci-run@v1
        with:
          target-instance: ${{ vars.SF_INSTANCE_URL }}
          action: |
            {"action":"task_execute_anon","apex_file":"scripts/apex/seed-accounts.apex"}
```

(The checkout is only for `cumulusci.yml`; the Apex is read at the signed
commit through the GitHub API.)

## Changes from the previous cci-run

- **Anonymous Apex is committed, and permitted by your policies.**
  `task_execute_anon` takes `apex_file` instead of `apex`; the
  `approved-digests` input, the per-run read of your GitHub Environment's
  protection and the `actions: read` permission are gone. Write a permit in
  `.busbar/policies/*.cedar` instead. Nothing checks Environment or
  default-branch protection.
- **Every run applies your repository's policies**, so every job needs
  `permissions: contents: read`, and `github-token` is always passed.
- **CumulusCI is an input.** The new `requirements` input takes a
  hash-locked requirements file (this action's `cci-lock.txt` by default);
  the install also contacts `pypi.org`. There is no API version window: any
  canonical `api_version` is accepted, and an unset one means the installed
  CumulusCI's default.
- **Requests are decided by effect**, not against one CumulusCI release's
  exact bytes and headers.
- **Unknown headers are removed, not refused.** A header the proxy does not
  forward (for example `Sforce-Enable-PKChunking`, a cookie or a tracing
  header) used to fail the request with `B5`; it is now removed before the
  policy sees the request, and the request goes on without it. List a
  header in `.busbar/door.toml` on your default branch to have it
  forwarded.
- **More SOQL**: `FIELDS()`, functions, `WITH`, `USING SCOPE` (within what
  Bulk API 1.0 supports), every field any clause names resolved.
- New outputs `risk` and `cumulusci-version`; the report carries the run
  context. `expect-denial` gains `approval-class` and `policy` and loses
  `needs-approval`.
- Earlier changes, still true: no `cci-args`, `org-name`, `cumulusci-version`,
  `allow-hosts`, `keychain-package`, `keychain-class` or `allowlist` inputs;
  `result_file` relative to `working-directory`; only `cumulusci.yml` is read
  from your project; the sandbox denies UDP (including DNS), raw and non-IP
  sockets, `io_uring` and `AF_UNIX` stream sockets, and the proxy serves only
  the org's API host.

## Platform requirements

The published binary is Linux x86_64 only, and `cci-run install` refuses any
interpreter other than CPython 3.12 on linux x86_64 (glibc 2.28 or later).
Installing needs `pypi.org` and `files.pythonhosted.org` to be reachable
from the runner; every run needs `api.github.com`.
