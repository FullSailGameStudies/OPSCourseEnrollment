
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

The workflow checks each student's Full Sail email against `hash.db` before provisioning repositories. The file must contain one SHA-256 hash per line, where each hash is the SHA-256 of the student's normalized email (trimmed and lowercased).

### Adding a single student

```bash
# Trim and lowercase the email, then hash and append to hash.db
printf '%s' "studentname@student.fullsail.edu" | tr -d '\r' | sed -e 's/^[[:space:]]*//' -e 's/[[:space:]]*$//' | tr '[:upper:]' '[:lower:]' | sha256sum | awk '{print $1}' >> hash.db
```

### Bulk adding students from a file

Create a plain text file (e.g. `students.txt`) with one email per line, then run:

```bash
while IFS= read -r email; do
  printf '%s' "$email" | tr -d '\r' | sed -e 's/^[[:space:]]*//' -e 's/[[:space:]]*$//' | tr '[:upper:]' '[:lower:]' | sha256sum | awk '{print $1}' >> hash.db
done < students.txt
```

### Notes

- Emails are normalized (trimmed of whitespace and lowercased) before hashing, so the input file does not need to be perfectly formatted.
- Each line in `hash.db` is a 64-character hex SHA-256 digest — no labels, no spaces.
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

## Deleting student repositories

Repositories are named with the pattern `OPS_<MONTH>_Lab<N>_<username>` (e.g. `OPS_JUN_Lab3_jdoe`). To delete all repos for a given month, filter by the `ops-student` topic and match the month prefix:

```bash
# Preview repos that will be deleted (dry run)
gh repo list FullSailGameStudies --topic ops-student --json name --jq '.[].name' | grep '^OPS_JUN_Lab'

# Delete all OPS_JUN_Lab* repos
gh repo list FullSailGameStudies --topic ops-student --json name --jq '.[].name' | grep '^OPS_JUN_Lab' | xargs -I {} gh repo delete "FullSailGameStudies/{}" --yes
```

Replace `JUN` with the three-letter month abbreviation you want to clean up (e.g. `JUL`, `AUG`, `SEP`).

> **Warning:** `gh repo delete --yes` skips the confirmation prompt. Always run the dry-run command first to verify which repos will be removed. Deletion is permanent and cannot be undone.

## Deleting closed issues

Student enrollment issues accumulate over time. Use the GitHub CLI to list and delete closed issues in this repository:

```bash
# List closed issues (number and title)
gh issue list --repo FullSailGameStudies/OPSCourseEnrollment --state closed --json number,title --jq '.[] | "\(.number)\t\(.title)"'

# Count closed issues
gh issue list --repo FullSailGameStudies/OPSCourseEnrollment --state closed --json number --jq 'length'

# Delete all closed issues
gh issue list --repo FullSailGameStudies/OPSCourseEnrollment --state closed --json number --jq '.[].number' | xargs -I {} gh issue delete {} --repo FullSailGameStudies/OPSCourseEnrollment --yes
```

> **Warning:** `gh issue delete --yes` skips the confirmation prompt. Always review the list of closed issues before deleting. Deletion is permanent and cannot be undone.
