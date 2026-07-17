Bulk Azure DevOps Organization Deletion Skill
> **Status:** Planning / implementation guidance for a controlled, resumable, UI-driven deletion workflow.
>
> **Scope:** Bulk deletion of Azure DevOps organizations using direct organization endpoints and interactive authentication.
---
Disclaimer of Warranty
THIS SCRIPT, SKILL, AND DOCUMENTATION ARE PROVIDED "AS IS" WITHOUT WARRANTY OF ANY KIND, EITHER EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE IMPLIED WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE, OR NON-INFRINGEMENT.
THE ENTIRE RISK ARISING OUT OF THE USE OR PERFORMANCE OF THIS SCRIPT, SKILL, AND DOCUMENTATION REMAINS WITH YOU. IN NO EVENT SHALL THE COPYRIGHT OWNERS OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES, INCLUDING BUT NOT LIMITED TO PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES, LOSS OF USE, DATA, PROFITS, OR BUSINESS INTERRUPTION, HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT, INCLUDING NEGLIGENCE OR OTHERWISE, ARISING IN ANY WAY OUT OF THE USE OF THIS SCRIPT, SKILL, OR DOCUMENTATION, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
---
Overview
This repository hosts an AI skill and supporting planning guidance for a bulk Azure DevOps organization deletion workflow.
The intended scenario is a customer-controlled cleanup of a large number of unused Azure DevOps organizations. The validated design uses browser automation against Azure DevOps organization settings pages, with interactive user authentication and checkpoint-based progress tracking.
The workflow was validated in a test environment using Playwright-driven Chromium in headed mode, with the user signing in interactively at runtime.
> [!WARNING]
> This workflow performs destructive actions at scale. Azure DevOps organization deletion enters a 28-day recoverable soft-delete state, but it is still a high-impact operation. Use dry-run validation, access verification, and formal approval before any live execution.
---
Validated Behavior
Testing confirmed the following behavior in Azure DevOps:
Azure DevOps organization URLs are deterministic:
Organization root: `https://dev.azure.com/{org-name}`
Organization settings overview: `https://dev.azure.com/{org-name}/_settings/organizationOverview`
The Delete organization button on the organization Overview page is the practical access signal for deletion rights.
Button present: the signed-in credential appears to have sufficient Owner/admin rights.
Button absent: the signed-in credential does not appear to have the required delete rights.
The delete flow requires:
Opening the organization Overview page.
Clicking Delete in the Delete organization section.
Confirming the warning dialog.
Typing the exact organization name into the confirmation field.
Clicking the final Delete confirmation button.
Successful deletion should be verified by requesting the organization Overview endpoint and confirming it returns HTTP 404 with a resource-not-found response.
The Azure DevOps landing/account list may briefly show stale cached organization entries after deletion. Do not use the landing list as the authoritative success signal.
Deletion is a 28-day recoverable soft-delete, not immediate permanent removal.
---
Customer Requirements
ID	Requirement
R1	Input file is named `organizations.csv`. The first column contains the organization name.
R2	Iterate over each organization and delete it through its direct endpoint: `https://dev.azure.com/{org-name}`.
R3	Process organizations in batches of 10.
R4	Write progress to `organizations_deleted.csv`.
R5	Do not use the test landing page `aex.dev.azure.com`.
R6	Do not hard-code, default, or store credentials. The user must sign in interactively at runtime.
R7	Run continuously with no pause between successful batches; flush progress after every batch of 10; pause on genuine errors.
R8	The process must be resumable and continue from prior progress on re-run.
---
Input File
Required file
```text
organizations.csv
```
Required column behavior
The first column must contain the Azure DevOps organization name.
The inspected file includes a header row and the header must be skipped.
The confirmed first-column header is:
```text
Organization
```
Inspected file assessment
The inspected production input file contained:
Attribute	Value
Total organizations	5,208
Header row	Yes
Duplicates	0
Empty / whitespace organization names	0
Organization names with spaces	0
Category	All 5,208 marked `Empty`
Organizations with projects	0
Full column set:
```text
Organization, Category, Region, Users, External users, External %, Projects, Repos,
Secret protection %, Repos disabled, Repos disabled %, Disablement state,
Admin access, Last user login, Comm status
```
---
Access Filtering
The inspected `organizations.csv` includes an `Admin access` column that pre-identifies whether the signed-in credential is expected to have deletion rights.
`Admin access` value	Count	Handling
`yes`	4,774	Eligible for deletion processing.
blank	431	Not expected to be deletable. Skip or remove before run.
`lost`	3	Not expected to be deletable. Skip or remove before run.
The three organizations marked `lost` were:
```text
REDACTED
REDACTED
REDACTED
```
Final handling decision
The user will pre-remove the 434 non-`yes` organizations before live execution. As a safety net, if any non-`yes` or no-access organization is still encountered, the workflow should record it as:
```text
SKIPPED_NO_ACCESS
```
This is a predictable skip and should not pause the run.
---
Output Files
`organizations_deleted.csv`
This is the primary progress and resume checkpoint file.
Recommended schema:
Column	Description
`org_name`	Organization name from input.
`org_url`	Organization root URL, for example `https://dev.azure.com/{org-name}`.
`batch_number`	1-based batch index where each batch contains up to 10 organizations.
`status`	`DELETED`, `ALREADY_ABSENT`, `SKIPPED_NO_ACCESS`, or `FAILED`.
`verified_404`	`Yes` or `No`. Indicates whether the authoritative post-delete 404 check passed.
`error`	Error or skip detail. Blank for clean success.
`timestamp_utc`	UTC timestamp when the outcome was recorded.
Optional `skipped_no_access.csv`
If desired, organizations skipped because of missing delete rights may also be emitted to a separate file:
```text
skipped_no_access.csv
```
This is optional because `organizations_deleted.csv` already supports `SKIPPED_NO_ACCESS` as a status.
---
Processing Design
The intended workflow is:
Launch a headed browser session.
Prompt the user to sign in interactively - this user should have the proper rights to delete the organizations - of course.
Read `organizations.csv`.
Skip the header row.
Build the organization worklist from the first column.
Reconcile prior progress from `organizations_deleted.csv`, if present.
Remove already completed organizations from the worklist.
Process the remaining organizations in batches of 10.
For each organization:
Navigate directly to `https://dev.azure.com/{org-name}/_settings/organizationOverview`.
If the endpoint already returns 404, record `ALREADY_ABSENT`.
If the delete control is absent, record `SKIPPED_NO_ACCESS`.
If the delete control is present, perform the confirmed delete workflow.
Verify success using the post-delete 404 check.
Append and flush results to `organizations_deleted.csv` after every batch of 10.
Continue without pausing between successful batches.
Pause only on genuine errors.
---
Resume Behavior
The workflow must be resumable by design.
`organizations_deleted.csv` is the single source of truth for checkpointing. On startup, the workflow should read the existing file and skip organizations already marked as:
```text
DELETED
ALREADY_ABSENT
SKIPPED_NO_ACCESS
```
Organizations marked as `FAILED`, or organizations that were in an interrupted in-flight batch and never flushed, should be retried.
Before retrying a delete, the workflow should check whether the organization Overview endpoint already returns 404. If it does, record:
```text
ALREADY_ABSENT
```
This makes recovery idempotent when an organization was deleted but the process stopped before the result was written to the progress file.
Progress must be appended or upserted by organization name. It must not be blindly overwritten or truncated during resume.
---
Error Handling
Non-pausing skip
Record `SKIPPED_NO_ACCESS` and continue when:
The organization Overview page loads, but the Delete organization button is absent.
The organization appears to lack admin/delete rights.
Genuine errors that should pause the run
Record `FAILED`, flush progress, and pause when:
The delete flow starts but the confirmation dialog does not appear.
The Type organization name confirmation field does not appear.
The final confirmation is clicked, but the organization endpoint does not return 404 within the expected timeout.
Authentication is lost mid-run.
Navigation repeatedly fails.
Any unexpected browser, UI, or service behavior occurs that is not covered by the no-access skip rule.
---
Safety Controls
Before live execution, confirm the following:
Formal customer approval exists for the deletion list.
The input file has been filtered to include only intended organizations.
Non-`yes` `Admin access` rows have been removed or are expected to be skipped.
A dry run has been completed.
The dry run confirms access and expected delete-button visibility.
The signed-in credential is the correct account.
The account has the intended directory context.
The process uses direct organization endpoints only.
The process does not use `aex.dev.azure.com`.
The progress file is being flushed after every batch of 10.
Recovery expectations are understood: deletion is recoverable for 28 days.
---
Recommended Dry-Run Mode
A dry-run mode is strongly recommended before live deletion.
Dry run should:
Read `organizations.csv` with only 4-6 orgs available.
Skip the header row.
Navigate to each organization Overview endpoint.
Check whether the organization already returns 404.
Check whether the Delete organization button is present.
Record what would happen without clicking delete.
Suggested dry-run statuses:
```text
WOULD_DELETE
ALREADY_ABSENT
SKIPPED_NO_ACCESS
FAILED_PRECHECK
```
---
Out of Scope
The stated customer requirement is UI automation using direct Azure DevOps organization endpoints.
Out of scope for this repository unless explicitly added later:
Azure DevOps organization management API implementation.
Credential storage or non-interactive credential injection.
Permanent deletion beyond the Azure DevOps soft-delete state.
Workflows that delete projects, repositories, users, or other resources independently of organization deletion.
---
---
Operational Notes
Use direct organization endpoints in the form `https://dev.azure.com/{org-name}/_settings/organizationOverview`.
Do not rely on the Azure DevOps account landing list as a success signal.
Use HTTP 404 from the organization Overview endpoint as the authoritative deletion verification.
Flush progress after every batch of 10.
Retry `FAILED` and unflushed organizations on resume.
Treat pre-existing 404 as `ALREADY_ABSENT`, not as an error.
Treat missing delete rights as `SKIPPED_NO_ACCESS`, not as an error.
Prompt for interactive sign-in at each run or resumed run.
Do not store credentials.
---
License
This project is licensed under the MIT License. See the LICENSE file for details.

The MIT License is a permissive open-source license that allows anyone to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the software, provided that the original copyright notice and license text are included in all copies or substantial portions of the software.
