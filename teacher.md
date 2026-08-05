
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

## Populating hash.db

The workflow checks each student's Full Sail email against `hash.db` before provisioning repositories. Each line in `hash.db` is formatted as:

```
<sha256_hash> <TAG>
```

where `<sha256_hash>` is the SHA-256 of the student's normalized email (trimmed and lowercased), and `<TAG>` is a cohort/section string (e.g. `AUG_S0`) that the workflow appends to the generated repository name. Repositories are named `OPS_Lab<N>_<TAG>_<username>` (e.g. `OPS_Lab3_AUG_S0_jdoe`).

### Adding a single student

```bash
# Trim and lowercase the email, then hash and append to hash.db with the given tag
TAG="AUG_S0"
printf '%s' "studentname@student.fullsail.edu" | tr -d '\r' | sed -e 's/^[[:space:]]*//' -e 's/[[:space:]]*$//' | tr '[:upper:]' '[:lower:]' | sha256sum | awk -v tag="$TAG" '{print $1, tag}' >> hash.db
```

### Bulk adding students from a file

Create a plain text file (e.g. `students.txt`) with one email per line, then run:

```bash
TAG="AUG_S0"
while IFS= read -r email; do
  printf '%s' "$email" | tr -d '\r' | sed -e 's/^[[:space:]]*//' -e 's/[[:space:]]*$//' | tr '[:upper:]' '[:lower:]' | sha256sum | awk -v tag="$TAG" '{print $1, tag}' >> hash.db
done < students.txt
```

### Notes

- Emails are normalized (trimmed of whitespace and lowercased) before hashing, so the input file does not need to be perfectly formatted.
- Each line in `hash.db` is `<64-character hex SHA-256 digest> <TAG>` — the tag is a single whitespace-delimited token (no spaces inside the tag).
- The `<TAG> MONTH_SECTION` becomes part of the provisioned repo name, so pick a stable, filesystem/GitHub-safe value (e.g. `AUG_S0`, `SEP_S1`). Avoid spaces and characters not allowed in GitHub repo names.
- Commit and push `hash.db` after updating it so the workflow can access it during runs.
- To remove a student, delete their hash line from `hash.db` and commit the change.

## Listing student repositories

Provisioned repositories are tagged with the topics `ops-student` and `not-graded`. Use the GitHub CLI (`gh`) to list them:

```bash
# List all repos in the org with both topics (table view)
gh repo list FullSailGameStudies --topic ops-student --topic not-graded

# List just the repo names
gh repo list FullSailGameStudies --topic ops-student --topic not-graded --json name --jq '.[].name'

# List with URLs
gh repo list FullSailGameStudies --topic ops-student --topic not-graded --json name,url --jq '.[] | "\(.name)\t\(.url)"'

# Count total provisioned repos
gh repo list FullSailGameStudies --topic ops-student --topic not-graded --json name --jq 'length'
```

> **Note:** `gh repo list` requires the `repo` scope. Authenticate with `gh auth login` if you haven't already.

## Cloning student repositories

Repositories are named with the pattern `OPS_Lab<N>_<TAG>_<username>` (e.g. `OPS_Lab3_AUG_S0_jdoe`), where `<TAG>` is the cohort/section string stored alongside each hash in `hash.db`. To clone all repos for a given tag, filter the `gh repo list` output by the tag and clone each match:

```bash
# Preview repos that will be cloned (dry run)
# gh repo list FullSailGameStudies --topic ops-student --topic not-graded --json url --jq '.[].url' | grep '^OPS_Lab3_AUG_S0_' # list via https url
gh repo list FullSailGameStudies --topic ops-student --topic not-graded --json sshUrl --jq '.[].sshUrl' | grep '^OPS_Lab[0-9]_AUG_S0_'

# Clone all OPS_Lab*_AUG_S0 repos into the current directory via SSH
gh repo list FullSailGameStudies --topic ops-student --topic not-graded --json sshUrl --jq '.[].sshUrl' | grep '^OPS_Lab[0-9]_AUG_S0_' | xargs -I {} git clone {}
```

Replace `AUG_S0` with the tag you want to clone (e.g. `AUG_S0`, `SEP_S1`).

> **Tip:** If you need to clone into a specific folder, create it first and run the command from inside that folder. Each repo will be cloned into its own subdirectory named after the repo.

## Deleting student repositories

Repositories are named with the pattern `OPS_Lab<N>_<TAG>_<username>` (e.g. `OPS_Lab3_AUG_S0_jdoe`), where `<TAG>` is the cohort/section string stored alongside each hash in `hash.db`. To delete all repos for a given tag, filter by the `ops-student` topic and match the tag:

```bash
# Preview repos that will be deleted (dry run)
gh repo list FullSailGameStudies --topic ops-student --json name --jq '.[].name' | grep '^OPS_Lab[0-9]_AUG_S0_'

# Delete all OPS_Lab*_AUG_S0 repos
gh repo list FullSailGameStudies --topic ops-student --json name --jq '.[].name' | grep '^OPS_Lab[0-9]_AUG_S0_' # | xargs -I {} gh repo # delete "FullSailGameStudies/{}" # --yes
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
