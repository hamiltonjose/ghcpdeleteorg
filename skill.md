
# Bulk Azure DevOps Organization Deletion skill

---

## 1. Background — what was proven in the test environment

A working, end-to-end organization-deletion flow was validated **in a test environment** using
Playwright-driven Chromium (headed, with interactive user authentication over CDP port 9222).

### Key findings 

1. **Org URL pattern is deterministic:**
   - Org root: `https://dev.azure.com/{org-name}`
   - Org settings overview (hosts the delete action): `https://dev.azure.com/{org-name}/_settings/organizationOverview`
2. **Access signal:** the presence of the **"Delete organization"** button on the Overview page
   indicates the credential has sufficient (Owner/admin) rights. Absence ⇒ insufficient
   (e.g., a plain *Member*).
3. **Delete flow (confirmed working):**
   - Click **Delete** in the "Delete organization" section → a confirmation dialog opens.
   - Dialog warns: *"All users will immediately lose access… up to 28 days to recover."*
   - A text field labeled **"Type organization name"** must be filled with the **exact org name**
     to enable the confirm **Delete** button.
   - Clicking confirm deletes the org and navigates the browser away (to the Azure DevOps
     products/home page).
4. **Authoritative success check:** after deletion, `https://dev.azure.com/{org-name}/_settings/organizationOverview`
   returns **HTTP 404 – "The resource cannot be found."**
   - ⚠️ The account landing list may briefly show a **stale cached** entry for a deleted org.
     Do **not** trust the landing list as the success signal — trust the **404** on the org
     endpoint.
5. **Soft delete:** deletion is a **28-day recoverable** state, not immediate permanent removal.

---

## 2. Scenario (full statement)

> The scenario is about the possibility to delete thousands of "garbage" Azure DevOps organizations**.

**Requirements as stated:**

| # | Requirement |
|---|-------------|
| R1 | Input is a file named **`organizations.csv`**. The **first column** holds the organization name. |
| R2 | Iterate over each org and delete it via its endpoint **`https://dev.azure.com/{org-name}`** (org endpoints used directly — **no separate landing/account URL**). |
| R3 | Process in **batches of 10** ("10 by 10"). |
| R4 | Report progress to a file named **`organizations_deleted.csv`**. |
| R5 | **Do not** use the test landing page (`aex.dev.azure.com`). |
| R6 | The credential **must not** be hard-coded / have a default. The tooling must **prompt the user** for the credential (interactive sign-in) at run time. |
| R7 | Run **continuously** (no pause between successful batches); **flush progress** to `organizations_deleted.csv` **after every batch of 10**; **pause on errors**. |
| R8 | The process must be **resumable**: if it stops (error or user intent), a later re-run **continues from where it left off**, not from the start. |

---

## 4. Data assessment — actual `organizations.csv`

The real input file was inspected. Findings:

| Attribute | Value |
|---|---|
| Total rows | **5,208 organizations** (+ 1 header row) |
| Header row | **Yes, present** — must be skipped |
| Column 1 | `Organization` (the org name) ✔ matches requirement |
| Batches of 10 | **521 batches** (520 × 10 + 1 × 8) |
| Duplicate org names | 0 |
| Empty / whitespace org names | 0 |
| Org names with spaces | 0 (names are 4–40 chars, URL-safe) |
| Category | **All 5,208 = "Empty"** (consistent with "garbage" orgs) |
| Projects > 0 | **0** (every org has zero projects) |

**Full column set:** `Organization, Category, Region, Users, External users, External %,
Projects, Repos, Secret protection %, Repos disabled, Repos disabled %, Disablement state,
Admin access, Last user login, Comm status`.

### 4.1 CRITICAL FINDING — the file pre-declares delete rights via `Admin access`

The **`Admin access`** column effectively pre-computes the access signal we otherwise had to probe
via the "Delete organization" button:

| `Admin access` | Count | Interpretation |
|---|---|---|
| `yes` | **4,774** | Credential is admin ⇒ **deletable** (sufficient) |
| *(blank)* | **431** | **No admin access** ⇒ delete will **fail / not be exposed** ⇒ would trigger a pause |
| `lost` | **3** | Access "lost"  ⇒ likely **not deletable** |

⚠️ **Note:** Only orgs with explicit Admin access=yes will be considered. Other cases will be skipped for processing 
and will be emitted into a separate `skipped_no_access.csv` for manual follow-up; 

---

## 5. Proposed design (for future implementation)

### 5.1 Inputs
- **`organizations.csv`** — first column = `Organization`. **Header row confirmed present → skip it.**
  Extra columns are ignored for deletion (but `Admin access` can drive pre-filtering, see §4.1).
- **Credential** — obtained via **interactive authentication** in a headed browser at run time.
  No credential is stored, defaulted, or embedded. (One sign-in covers the whole run, assuming
  all orgs live under the directory the signed-in account administers.)

### 5.2 Processing loop
1. Launch a headed browser; user signs in interactively (one time).
2. Read `organizations.csv`; build the work list of org names.
3. **Load prior progress** from `organizations_deleted.csv` (if it exists) and **subtract already-
   processed orgs** from the work list (see §5.6 Resume).
4. Iterate the **remaining** orgs in **chunks of 10**:
   - For each org in the chunk, navigate to `https://dev.azure.com/{org-name}/_settings/organizationOverview`
     and perform the validated delete flow (click Delete → type exact org name → confirm).
   - Verify each deletion via the **404** check on the org endpoint.
   - Record the per-org outcome.
   - **After each batch of 10**, append/flush results to `organizations_deleted.csv`.
5. **On error** (see §5.4), **pause** and surface the failure for the user to decide.
6. Continue until all orgs are processed.

### 5.3 Progress file — `organizations_deleted.csv` (proposed schema)
| Column | Meaning |
|--------|---------|
| `org_name` | Organization name from input |
| `org_url` | `https://dev.azure.com/{org-name}` |
| `batch_number` | 1-based batch index (each batch = up to 10 orgs) |
| `status` | `DELETED` / `ALREADY_ABSENT` (pre-existing 404) / `SKIPPED_NO_ACCESS` / `FAILED` |
| `verified_404` | `Yes` / `No` (post-delete authoritative check) |
| `error` | Error/skip detail when status = FAILED or SKIPPED_NO_ACCESS, else blank |
| `timestamp_utc` | When the outcome was recorded |

Flushing after every batch makes the run **resumable**: on restart, orgs already marked
`DELETED`/`verified_404=Yes` can be skipped.

> **Non-`yes` admin orgs (RESOLVED):** the user will **pre-remove** the 434 non-`yes` `Admin
> access` orgs from `organizations.csv` before the run. As a safety net, if any non-`yes` /
> no-delete-rights org is nonetheless encountered, the job **skips it and records
> `SKIPPED_NO_ACCESS`** (with the reason in `error`). This is **not** treated as an error and does
> **not** pause the run.

### 5.4 Error handling (per R7 — pause on errors)
**Distinguish a predictable no-access skip from a genuine error.**

- **Skip (no pause), status `SKIPPED_NO_ACCESS`:** the org shows no admin/delete rights — the
  **Delete organization** button is absent (the expected signature of a non-`yes` org). Record and
  move on.

Treat as a genuine **error** and **pause** only when, for a given org:
- The delete flow starts but the confirmation dialog / "Type organization name" field does not
  appear, or
- After confirming, the org endpoint does **not** return 404 within a timeout, or
- Navigation/authentication is lost mid-run, or
- Any unexpected failure not covered by the skip rule above.

On pause: record the failing org as `FAILED` with the reason, stop, and wait for the user to
inspect / re-authenticate / decide whether to skip and continue.

### 5.5 Safety considerations to confirm before any execution
- **Irreversibility at scale:** deletion is recoverable for 28 days, but deleting 5,208 orgs is
  high-impact. A **dry-run mode** (resolve + report the delete button's presence per org, without
  confirming) is strongly recommended before a live run.
- **Access pre-check:** the `Admin access` column already flags the **434 non-`yes` orgs**; use it
  to pre-filter and avoid predictable pauses.
- **Rate/throttling:** 5,208 UI-driven deletions may hit UI slowness or service throttling;
  batch pacing and generous timeouts will matter.

### 5.6 Resume from checkpoint (per R8)
The process must survive a stop — **error, user interruption (Ctrl-C / close), crash, or lost
auth** — and continue on a later re-run **without re-processing completed orgs**.

- **Progress file is the checkpoint.** `organizations_deleted.csv` is the single source of truth
  for what has already been handled. Because it is flushed **after every batch of 10**, at most one
  in-flight batch of work is ever unrecorded.
- **On startup, reconcile:** read `organizations_deleted.csv`, build the set of org names already
  marked `DELETED`, `ALREADY_ABSENT`, or `SKIPPED_NO_ACCESS`, and **remove them from the work
  list**. Only the remainder is processed. Running the job again after full completion is a no-op.
- **`FAILED` / interrupted orgs are retried.** Any org that is `FAILED`, or was in the unflushed
  in-flight batch when the stop happened, is **re-attempted** on resume. The delete flow is
  **idempotent by verification**: before deleting, check the org endpoint — if it already returns
  **404**, mark `ALREADY_ABSENT` and skip; otherwise proceed. This makes a re-run safe even if an
  org was actually deleted just before the crash but never recorded.
- **Append, don't overwrite.** Resume must **append** to the existing `organizations_deleted.csv`
  (or upsert by `org_name`), never truncate it. (Consider de-duplicating on `org_name`, keeping the
  latest outcome, if the same org is retried.)
- **Re-authentication on resume:** since credentials are never stored (R6), a resumed run begins
  with a fresh interactive sign-in; processing then continues from the reconciled remainder.
- **No separate state store needed:** the CSV plus the pre-delete 404 check fully determine resume
  position — no database or lock file is required, though a lightweight run-log is optional.

---

## 6. General guideline

1. **Single directory:** all orgs under one directory the signed-in credential administers, so
   a single interactive sign-in suffices
2. **Already-deleted / non-existent orgs:** treat a pre-existing 404 as `ALREADY_ABSENT` (not an
   error) and continue
3. ~~Non-`yes` Admin access rows: pre-filter or skip-inline?~~ **ANSWERED** — user pre-removes them
   from the input; if any are still encountered, **skip and record `SKIPPED_NO_ACCESS`** (no pause).
4. Use **org endpoints directly** (`https://dev.azure.com/{org-name}`); **no** separate
  landing/account URL. *(Do not use `aex.dev.azure.com`.)*
5. Run **continuously**; **flush progress every 10**; **pause on errors**.
6. **Prompt** for the credential interactively; **no default credential**.
7. **Input has a header row** (`Organization` = col 1) → skip it.
8. **Non-`yes` admin orgs** are **pre-removed by the user**; any encountered are **skipped and
  documented as `SKIPPED_NO_ACCESS`** — not an error, no pause.
9. **Resumable (R8):** if stopped for any reason, a re-run reconciles against
  `organizations_deleted.csv` and continues from where it left off; already-`DELETED` /
  `ALREADY_ABSENT` / `SKIPPED_NO_ACCESS` orgs are skipped, `FAILED`/in-flight orgs are retried
  idempotently (pre-delete 404 check). Progress file is appended, never overwritten.
