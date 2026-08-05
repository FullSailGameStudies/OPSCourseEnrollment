# Setup Guide

This guide walks you through the complete initial setup of a Course Enrollment system. Complete every section in order before accepting student enrollment requests.

---

## Prerequisites

- **GitHub Organization** -
   - https://github.com/FullSailGameStudies
- **GitHub account** that is an **owner** of the organization you're using for this (or has org admin + repo admin on all relevant repos)
- **GitHub CLI** installed and authenticated (`gh auth login`)
- **Git** installed and configured
- Admin access to create repositories and secrets in your organization

---

## 1. Create the Enrollment Repository

If this repository does not already exist in the organization:

```bash
gh repo create YourOrganizationName/RepositoryNameForCourseEnrollment
```

NOTE: the repo must be public so that students can access the Issues page.

Then push this project's code to it:

```bash
git remote add origin https://github.com/FullSailGameStudies/PG2CourseEnrollment.git
git push -u origin main
```

> If you cloned this repo from an existing remote, skip this step.

---

## 2. Create Template Repositories

The provision workflow creates student repos from templates. The template must:

1. **Exist** in your organization.
2. Be marked as a **template repository** (Settings > "Template repository" checkbox).

---

## 3: Create a GitHub App in Your Organization

   1. Go to your Organization Settings -> scroll down to Developer settings -> click GitHub Apps -> click New GitHub App.
   2. GitHub App Name: Give it a clear name (e.g., PG2 Enrollment Bot).
   3. Homepage URL: Put the organization's URL (e.g. https://online.fullsail.edu/)
   4. Uncheck Active under the Webhook section (you do not need webhooks).
   5. Scroll down to Permissions -> Repository permissions:
      * Administration: Read & Write (Required to create repos from templates)
      * Contents: Read and Write
      * Custom Properties: Read and Write
      * Discussions: Read and Write
      * Issues: Read and Write
      * Pull requests: Read and Write
      * Workflows: Read and Write
      * Metadata: Read-only (Automatically granted)
   6. Scroll down to Where can this GitHub App be installed? -> Select Only on this account (your organization).
   7. Click Create GitHub App.

------------------------------

## 4: Save the App Identification Tokens
Once created, your App management page will load. We need to grab two pieces of information:

   1. App ID: Copy the number listed near the top (e.g., 123456).
   2. Generate a Private Key: Scroll to the bottom and click Generate a private key. A file (ending in .pem) will automatically download to your computer. Open this file in notepad and copy the entire block of text.

------------------------------
## 5: Save the Tokens as Secrets in Your Repository
Go to your course-enrollment repository Settings -> Secrets and variables -> Actions -> create two repository secrets:

* Name: APP_ID | Value: Paste your numeric App ID.
* Name: APP_PRIVATE_KEY | Value: Paste the entire text content of your .pem file.

------------------------------
## 6: Install the App inside your Organization

   1. On your GitHub App settings page, click Install App on the left sidebar menu.
   2. Click Install next to your organization name.
   3. Restrict it to only selected repositories
      * select "Only select repositories"
      * then choose the course enrollment repo AND the template repo
   4. click "Save"


------------------------------
## 7: Add a Label to be used by the scripts

   1. On your Issues page of the enrollment repo, click the "Label" tab.
   2. Click the "New label" button.
   3. Add a name (e.g. provision-repo)
   4. Click the "Create label" button.
   5. Update the provision.yml and the student-invite.yml scripts with the name of the label.


## What to do each month
Read the [teacher.md](./teacher.md) for instructions the teachers must do each month.