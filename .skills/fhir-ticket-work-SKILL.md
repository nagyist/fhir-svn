---
name: fhir-ticket-work
description: >
  Work a FHIR JIRA ticket end-to-end: set up a git branch, read the ticket's resolution description,
  and make the required changes to the FHIR specification source files.
  Use this skill whenever the user says they want to work on a FHIR ticket, start a JIRA issue,
  pick up a tracker item, begin work on a FHIR-XXXXX issue, or anything that implies they're
  about to make changes to the FHIR spec for a specific ticket. Also trigger when the user
  mentions a FHIR ticket number (like FHIR-54554) in the context of wanting to work on it,
  fix it, or apply changes for it.
---

# FHIR Ticket Work

This skill handles working a FHIR JIRA ticket: setting up a git branch, reading the ticket's resolution description, and making the required changes to the FHIR specification source files.

## Context

The user works on the HL7 FHIR specification, which lives in a git repo at `HL7/fhir` on GitHub. When they pick up a JIRA ticket from jira.hl7.org, the first thing they need is a clean branch to work on. The branch naming convention ensures changes are traceable back to the person and the ticket.

## Prerequisites

- The working directory must be the fhir repo (the user's mounted workspace folder).
- The repo should be a clone of `https://github.com/HL7/fhir.git`.

## Workflow

### Step 1: Identify the ticket

Figure out which ticket the user wants to work on. They might say it in various ways:

- "I want to work on FHIR-54554"
- "Let me pick up that DiagnosticReport conclusionCode ticket"
- "Start work on 54554"
- A ticket key from a previous JIRA lookup in the conversation

You need the **FHIR ticket key** (e.g., `FHIR-54554`). If the user gives just a number, prepend `FHIR-`. If they describe the ticket by name and you can identify it from earlier in the conversation (e.g., from the hl7-jira-filter skill results), use that.

### Step 2: Ensure a clean working tree

Before creating the branch, make sure the working tree is clean:

```bash
git status --porcelain
```

If there are uncommitted changes, let the user know and ask how they'd like to handle them (stash, commit, or discard). Don't proceed with a dirty working tree — the user might lose work.

### Step 3: Update master

Pull the latest from origin so the branch starts from a current base:

```bash
git checkout master
git pull origin master
```

If the user is already on master and it's up to date, this is quick. If they're on another branch, switching to master first ensures the new branch has a clean starting point.

### Step 4: Create or checkout the branch

The branch name follows this convention:

```
jdlnolen-{TICKET_KEY}
```

The ticket key should be uppercase (e.g., `FHIR-54554`, not `fhir-54554`).

First, check whether the branch already exists (locally or on the remote):

```bash
git branch --list "jdlnolen-FHIR-54554"
git branch -r --list "origin/jdlnolen-FHIR-54554"
```

**If the branch does not exist** — create it from the freshly updated master:

```bash
git checkout -b jdlnolen-FHIR-54554
```

**If the branch already exists locally** — check it out and bring it up to date. The goal is to make sure the branch has the latest changes from both the remote copy of itself (if someone pushed to it) and from master (so it doesn't drift too far behind):

```bash
git checkout jdlnolen-FHIR-54554
git pull origin jdlnolen-FHIR-54554 --ff-only 2>/dev/null || true
git merge master --no-edit
```

The `pull --ff-only` fetches any new commits pushed to the remote branch. It's wrapped with `|| true` because the remote branch may not exist yet (e.g., if the user hasn't pushed). The merge from master keeps the branch current with the latest spec changes.

If the merge has conflicts, let the user know and list the conflicting files. Don't try to resolve them automatically — the user needs to decide how to handle spec conflicts.

**If the branch exists only on the remote** — fetch it and set up tracking:

```bash
git checkout -b jdlnolen-FHIR-54554 origin/jdlnolen-FHIR-54554
git merge master --no-edit
```

### Step 5: Confirm

Tell the user the branch is ready. Adjust the message depending on what happened:

- **New branch**: "You're on branch `jdlnolen-FHIR-54554`, branched from the latest `master`. Ready to start."
- **Existing branch, now updated**: "You're back on branch `jdlnolen-FHIR-54554`. I've pulled the latest and merged in current `master` — you're up to date."
- **Merge conflicts**: "You're on branch `jdlnolen-FHIR-54554`, but there are merge conflicts with master in: [list files]. You'll need to resolve those before continuing."

If you have context about what the ticket is about (from a previous JIRA lookup), briefly remind them — it saves them from having to look it up again.

---

## Phase 2: Read the Ticket and Make Changes

Once the branch is set up, the next phase is understanding what needs to change and making the edits.

### Step 6: Read the ticket's resolution description

Navigate to the JIRA ticket using browser automation (Claude in Chrome) to get the full resolution description. This is the authoritative statement of what changes need to be made.

1. Use `tabs_context_mcp` to check for existing tabs. Reuse a jira.hl7.org tab if one is open, otherwise create a new one.
2. Navigate to `https://jira.hl7.org/browse/{TICKET_KEY}` (e.g., `https://jira.hl7.org/browse/FHIR-54554`).
3. Wait 3 seconds for the page to load, then use `get_page_text` to extract the ticket content.
4. From the page text, extract:
   - **Resolution Description** — this is the key field. It describes exactly what changes to make. Look for the text after "Resolution Description:" in the page output.
   - **Resolution** — e.g., "Persuasive with Modification", "Persuasive", "Not Persuasive". This confirms the ticket was approved.
   - **Related Artifact(s)** — which FHIR resource(s) are affected (e.g., DiagnosticReport, Observation).
   - **Description** — the original issue description, for additional context.

If the page requires login, ask the user to log in and let you know when they're done.

### Step 7: Read the spec editing guide

Before making changes, read the CLAUDE.md file at the root of the fhir repo:

```bash
cat CLAUDE.md
```

This is the authoritative reference for the repo's file structure, naming conventions, and editing patterns. Read it in full — the FHIR spec source is organized in a specific way and the build process depends on files being in the right format.

If CLAUDE.md doesn't exist or can't be read, the essential knowledge is summarized below. But always prefer the live file — it may have been updated since this skill was written.

#### Quick Reference: Source File Structure

All source content lives under `source/` organized by resource name (lowercase):

```
source/{resource-name}/
├── structuredefinition-{Resource}.xml    # Formal resource definition (the main one)
├── bundle-{Resource}-search-params.xml   # Search parameter definitions
├── {resource}-spreadsheet.xlsx           # Resource structure spreadsheet
├── {resource}-introduction.xml           # Narrative intro (scope & usage)
├── {resource}-notes.xml                  # Additional implementation notes
├── {resource}-example*.xml               # Example instances
├── list-{Resource}-examples.xml          # Registry of examples
├── list-{Resource}-operations.xml        # Available operations
├── valueset-*.xml                        # Related value sets
├── codesystem-*.xml                      # Related code systems
└── implementationguide-{Resource}-core.xml
```

Note the casing: directory and filenames use lowercase `{resource-name}`, but XML element references and some filenames use PascalCase `{Resource}` (e.g., `source/diagnosticreport/structuredefinition-DiagnosticReport.xml`).

#### Key Editing Patterns

**Structure definition changes** → edit `structuredefinition-{Resource}.xml`. When modifying elements (renaming, adding, changing types), always also check and update the corresponding search parameters in `bundle-{Resource}-search-params.xml`. Search parameter expressions must match actual element paths.

**Search parameter changes** → edit `bundle-{Resource}-search-params.xml`. Each entry is an `<entry>` containing a `<SearchParameter>` resource with id, code, type, and FHIRPath expression.

**CodeableReference elements** require **two** search parameters:
- Reference search: `[Resource].element.reference` with type `reference`
- Token search: `[Resource].element.concept` with type `token`
Do NOT use `type="special"` or `type="composite"` for CodeableReference.

**Narrative/documentation changes** → edit `{resource}-introduction.xml` (scope and usage) or `{resource}-notes.xml` (implementation notes). These are XHTML fragments.

**New examples** → create `{resource}-example-{name}.xml`, then register it in `list-{Resource}-examples.xml`.

**Code system changes** (e.g., request-intent) → find the relevant spreadsheet XML (like `source/request/request-spreadsheet.xml`). These use Excel XML format with hierarchical codes.

**Bulk text replacements** → use Python over `sed` for reliability, especially on macOS:
```python
import glob
for fp in glob.glob(pattern, recursive=True):
    with open(fp, 'r') as f: content = f.read()
    if 'old' in content:
        with open(fp, 'w') as f: f.write(content.replace('old', 'new'))
```

**Text search** → use `grep` for finding things across the repo.

**XML validation** → after edits, validate with `xmllint --noout {file}`.

### Step 8: Understand the current state of the spec

Before making changes, look at the current published spec to understand what exists today. The official build is at **build.fhir.org**.

For example, if the ticket affects DiagnosticReport, check:
- `https://build.fhir.org/diagnosticreport.html` — the resource page
- Use browser automation to navigate there and read the relevant sections

This gives you context for what the resolution description is asking you to change. Compare the current published spec with what the resolution says it should become.

Also look at the actual source files that will need to change. Use `grep` to find relevant content:
```bash
grep -r "element-name" source/{resource-name}/
```

### Step 9: Make the changes

Apply the changes described in the resolution description to the spec source files.

General principles:
- **NEVER edit anything in the `publish/` directory** — this is the generated output folder. It is rebuilt by the publisher and any manual changes will be overwritten. HTML files under `source/` (e.g., module pages) are legitimate source templates and may be edited.
- Follow the resolution description precisely — it was voted on and approved by the work group.
- If the resolution description is ambiguous, ask the user for clarification rather than guessing.
- Make minimal changes — only what the resolution calls for, nothing more.
- Keep the existing code style and formatting conventions of whichever file you're editing.
- When renaming elements, update BOTH the structure definition AND search parameters.
- Validate XML after changes: `xmllint --noout {file}`

After making changes, briefly summarize what you changed and in which files so the user can review.

### Step 10: Return to master

Once the changes are complete, switch back to master so the repo is ready for the next ticket:

```bash
git checkout master
```

This keeps the working directory in a clean state for the next `git checkout -b` when picking up another ticket. The changes on the ticket branch are preserved — they'll be there when the user pushes or revisits the branch later.
