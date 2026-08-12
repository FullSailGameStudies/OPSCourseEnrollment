
# Teacher Guide



```bash
# Generate a new passcode
echo -n "your-passcode" | sha256sum
# OR
printf '%s' "your-passcode" | sha256sum
```


## Updating the passcode

1. Generate a new passcode hash
2. Update the `EXPECTED_HASH` variable in the workflow file .github/workflows/provision.yml
3. Commit and push the changes
4. Test the workflow by creating a new issue with the new passcode
5. Share the new passcode with students

## Populating hash.db/HASHDB repository variable

The workflow checks each student's Full Sail email against the `HASHDB` **repository variable** (configured under Settings → Secrets and variables → Actions → Variables tab → repository variables section) before provisioning repositories. Each line in `HASHDB` is formatted as:

```
<sha256_hash> <TAG>
```

where `<sha256_hash>` is the SHA-256 of the student's normalized email (trimmed, lowercased, and with the last 3 characters stripped so `.com` and `.edu` Full Sail addresses hash identically), and `<TAG>` is a cohort/section string (e.g. `AUG_S0`) that the workflow appends to the generated repository name. Repositories are named `OPS_Lab<N>_<TAG>_<username>` (e.g. `OPS_Lab3_AUG_S0_jdoe`).

> **Why a variable instead of a file?** Storing the hashes in a repository variable keeps the authorized-student list out of the git history and lets you update it without committing. The workflow reads it via `${{ vars.HASHDB }}` and greps it the same way it used to grep `hash.db`.

### Configuring the variable in GitHub

You can set or update `HASHDB` either through the GitHub web UI or with the GitHub CLI.

**Option A — Web UI**

1. Go to the repository on GitHub: **Settings → Secrets and variables → Actions → Variables**.
2. Click **New repository variable**.
3. Name: `HASHDB`
4. Value: paste the full list of `<sha256_hash> <TAG>` lines (one per line).
5. Click **Add variable**.

To update later, click the existing `HASHDB` entry, edit the value, and save.

**Option B — GitHub CLI**

```bash
# Set HASHDB from the contents of a local file (e.g. hash.db or students_hashes.txt)
gh variable set HASHDB --body "$(cat hash.db)"

# Or pipe hashes in directly from a generation command (see examples below)
gh variable set HASHDB --body "$(cat <<'EOF'
<sha256_hash> <TAG>
<sha256_hash> <TAG>
EOF
)"

# View the current value
gh variable get HASHDB

# Delete the variable
gh variable delete HASHDB
```

> **Note:** `gh variable set` overwrites the existing value. To append new students, first retrieve the current value with `gh variable get HASHDB`, append the new lines, and set the combined value back.

### Adding a single student

```bash
# Trim and lowercase the email, strip the last 3 chars (TLD) so .com/.edu match, then hash.
# Prints "<hash> <TAG>" — copy this line into the HASHDB variable value.
TAG="AUG_S0"
printf '%s' "studentname@student.fullsail.edu" | tr -d '\r' | sed -e 's/^[[:space:]]*//' -e 's/[[:space:]]*$//' | tr '[:upper:]' '[:lower:]' | sed 's/...$//' | sha256sum | awk -v tag="$TAG" '{print $1, tag}' >> hash.db
```

### Bulk adding students from a file

Create a plain text file (e.g. `students.txt`) with one email per line, then run:

```bash
TAG="AUG_S1"
# Normalize all emails once (strip CR, trim whitespace, lowercase, strip last 3 chars
# so .com/.edu match), then hash each line and print "<hash> <TAG>".
# Pipe the output into `gh variable set` to update HASHDB in one step.
tr -d '\r' < students.txt \
  | sed -e 's/^[[:space:]]*//' -e 's/[[:space:]]*$//' \
  | tr '[:upper:]' '[:lower:]' \
  | sed 's/...$//' \
  | while IFS= read -r email; do
      printf '%s' "$email" | sha256sum | awk -v tag="$TAG" '{print $1, tag}' >> hash.db
    done
```

To merge these new lines with the existing authorized list and push the result back to GitHub:

```bash
NEW_HASHES="$(tr -d '\r' < students.txt \
  | sed -e 's/^[[:space:]]*//' -e 's/[[:space:]]*$//' \
  | tr '[:upper:]' '[:lower:]' \
  | sed 's/...$//' \
  | while IFS= read -r email; do
      printf '%s' "$email" | sha256sum | awk -v tag="$TAG" '{print $1, tag}'
    done)"

gh variable set HASHDB --body "$(printf '%s\n%s\n' "$(gh variable get HASHDB)" "$NEW_HASHES")"
```

### Bulk adding students from a Full Sail roster CSV (csvkit)

Full Sail roster exports (e.g. `AUG_S1_Roster.csv`) are CSV files with a `Primary Email` column. Use [csvkit](https://csvkit.readthedocs.io/) (`brew install csvkit`) to extract that column, then feed each email through the same normalization + hashing pipeline used above.

```bash
# Preview the emails that will be hashed (requires: brew install csvkit)
csvcut -c "Primary Email" AUG_S1_Roster.csv | grep -v "Primary Email" | sort
```

```bash
# Extract emails from the roster CSV, normalize the whole stream once (strip CR, trim
# whitespace, lowercase, strip last 3 chars so .com/.edu match), then hash each line and
# append "<hash> <TAG>" to hash.db
TAG="AUG_S1"
csvcut -c "Primary Email" AUG_S1_Roster.csv \
  | grep -v "Primary Email" \
  | grep -v "@fullsail." \
  | tr -d '\r' \
  | sed -e 's/^[[:space:]]*//' -e 's/[[:space:]]*$//' \
  | tr '[:upper:]' '[:lower:]' \
  | sed 's/...$//' \
  | while IFS= read -r email; do
      printf '%s' "$email" | sha256sum | awk -v tag="$TAG" '{print $1, tag}' >> hash.db
    done
```

> **Tip:** Run the preview command first to confirm the column name and that the emails look correct before updating `HASHDB`. If your roster uses a different column name (e.g. `Email`, `Student Email`), adjust the `-c` argument accordingly.

### Notes

- Emails are normalized (trimmed of whitespace, lowercased, and stripped of the last 3 characters so `.com` and `.edu` Full Sail addresses produce the same hash) before hashing, so the input file does not need to be perfectly formatted.
- Each line in `HASHDB` is `<64-character hex SHA-256 digest> <TAG>` — the tag is a single whitespace-delimited token (no spaces inside the tag).
- The `<TAG> MONTH_SECTION` becomes part of the provisioned repo name, so pick a stable, filesystem/GitHub-safe value (e.g. `AUG_S0`, `SEP_S1`). Avoid spaces and characters not allowed in GitHub repo names.
- After updating `HASHDB` the workflow picks up the new value on its next run — no commit or push required.
- To remove a student, edit the `HASHDB` variable (via the web UI or `gh variable set`) and delete their hash line.

## Listing student repositories

Provisioned repositories are tagged with the topics `ops-student` and `not-graded`. Use the GitHub CLI (`gh`) to list them:

```bash
# List all repos in the org with both topics (table view)
gh repo list FullSailGameStudies -L 1000 --topic ops-student --topic not-graded

# List just the repo names
gh repo list FullSailGameStudies -L 1000 --topic ops-student --topic not-graded --json name --jq '.[].name'

# List with URLs
gh repo list FullSailGameStudies -L 1000 --topic ops-student --topic not-graded --json name,url --jq '.[] | "\(.name)\t\(.url)"'

# Count total provisioned repos
gh repo list FullSailGameStudies -L 1000 --topic ops-student --topic not-graded --json name --jq 'length'
```

> **Note:** `gh repo list` requires the `repo` scope. Authenticate with `gh auth login` if you haven't already.
>
> **Pagination:** `gh repo list` only returns the first 30 repositories by default. The `-L 1000` flag raises that limit so every provisioned repo is returned in a single call — without it, cohorts larger than 30 students would be silently truncated. If a cohort ever exceeds 1000 repos, bump the value higher.

## Cloning student repositories

Repositories are named with the pattern `OPS_Lab<N>_<TAG>_<username>` (e.g. `OPS_Lab3_AUG_S0_jdoe`), where `<TAG>` is the cohort/section string stored alongside each hash in the `HASHDB` repository variable. To clone all repos for a given tag, filter the `gh repo list` output by the tag and clone each match:

```bash
# Preview repos that will be cloned (dry run)
# gh repo list FullSailGameStudies -L 1000 --topic ops-student --topic not-graded --json url --jq '.[].url' | grep '^OPS_Lab3_AUG_S0_' # list via https url
gh repo list FullSailGameStudies -L 1000 --topic ops-student --topic not-graded --json sshUrl --jq '.[].sshUrl' | grep '^OPS_Lab[0-9]_AUG_S0_'

# Clone all OPS_Lab*_AUG_S0 repos into the current directory via SSH
gh repo list FullSailGameStudies -L 1000 --topic ops-student --topic not-graded --json sshUrl --jq '.[].sshUrl' | grep '^OPS_Lab[0-9]_AUG_S0_' | xargs -I {} git clone {}
```

Replace `AUG_S0` with the tag you want to clone (e.g. `AUG_S0`, `SEP_S1`).

> **Tip:** If you need to clone into a specific folder, create it first and run the command from inside that folder. Each repo will be cloned into its own subdirectory named after the repo.

## Deleting student repositories

Repositories are named with the pattern `OPS_Lab<N>_<TAG>_<username>` (e.g. `OPS_Lab3_AUG_S0_jdoe`), where `<TAG>` is the cohort/section string stored alongside each hash in the `HASHDB` repository variable. To delete all repos for a given tag, filter by the `ops-student` topic and match the tag:

```bash
# Preview repos that will be deleted (dry run)
gh repo list FullSailGameStudies -L 1000 --topic ops-student --json name --jq '.[].name' | grep '^OPS_Lab[0-9]_AUG_S0_'

# Delete all OPS_Lab*_AUG_S0 repos
gh repo list FullSailGameStudies -L 1000 --topic ops-student --json name --jq '.[].name' | grep '^OPS_Lab[0-9]_AUG_S0_' # | xargs -I {} gh repo # delete "FullSailGameStudies/{}" # --yes
```

Replace `AUG_S0` with the tag you want to clean up (e.g. `AUG_S0`, `SEP_S1`).

> **Warning:** `gh repo delete --yes` skips the confirmation prompt. Always run the dry-run command first to verify which repos will be removed. Deletion is permanent and cannot be undone.

## Deleting closed issues

Student enrollment issues accumulate over time. Use the GitHub CLI to list and delete closed issues in this repository:

```bash
# List closed issues (number and title)
gh issue list --repo FullSailGameStudies/OPSCourseEnrollment --state closed --json number,title --jq '.[] | "\(.number)\t\(.title)"'

# Count closed issues
gh issue list --repo FullSailGameStudies/OPSCourseEnrollment --state closed --json number --jq 'length'

# Delete all closed issues
gh issue list --repo FullSailGameStudies/OPSCourseEnrollment --state closed --json number --jq '.[].number' | xargs -I {} gh issue # delete {} --repo FullSailGameStudies/OPSCourseEnrollment # --yes
```

> **Warning:** `gh issue delete --yes` skips the confirmation prompt. Always review the list of closed issues before deleting. Deletion is permanent and cannot be undone.

<details>
<summary>Marking repositories as graded (adding <code>graded</code>, removing <code>not-graded</code>)</summary>

The following bash function marks a student repository as graded by adding the `graded` topic and removing the `not-graded` topic. With no argument, it operates on the repo in the current directory (resolved via `gh repo view`). Add it to your shell(.bashrc or .zshrc), then call it with zero or one repo name.

```bash
# Mark a student repo as graded: add `graded`, remove `not-graded`.
# With no arg, uses the current directory's repo.
# Usage: fs_set_graded [OPS_Lab3_AUG_S0_jdoe]
fs_set_graded() {
  local repo="${1:-$(gh repo view --json nameWithOwner --jq '.nameWithOwner' 2>/dev/null | sed 's#^[^/]*/##')}"
  [ -n "$repo" ] || { echo "not in a GitHub repo and no repo name given" >&2; return 2; }
  echo "marking FullSailGameStudies/${repo} as graded"
  gh api -X PUT    "repos/FullSailGameStudies/${repo}/topics/graded" --silent \
    && gh api -X DELETE "repos/FullSailGameStudies/${repo}/topics/not-graded" --silent \
    && echo "  done: ${repo} is now graded"
}
```

> **Note:** Requires the `repo` scope. Authenticate with `gh auth login` if you haven't already. To mark every `not-graded` repo for a given tag at once, pipe a `gh repo list` query into the function:
>
> ```bash
> # Mark all AUG_S0 repos as graded
> gh repo list FullSailGameStudies -L 1000 --topic ops-student --topic not-graded \
>   --json name --jq '.[].name' \
>   | grep '^OPS_Lab[0-9]_AUG_S0_' \
>   | xargs -I {} fs_set_graded {}
> ```

</details>
