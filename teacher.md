
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
# Hash a student's email and append it to hash.db
printf '%s' "studentname@student.fullsail.edu" | sha256sum | awk '{print $1}' >> hash.db
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

