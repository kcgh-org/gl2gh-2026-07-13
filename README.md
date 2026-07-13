# GitLab → GitHub Migration

## 1. Executive Summary – Objective
This document provides detailed procedures to migrate source code repositories from **GitLab Server** to **GitHub**.

### 1.2 Migration Execution Approaches

GitLab projects can be migrated to GitHub using one of the following approaches. Both approaches support migrations to GitHub Enterprise Cloud and GitHub Enterprise Cloud with Data Residency.

#### a. Manual GEI Migration
- Execute migration scripts manually on a supported Linux host.
- The process consists of:
  - Perform migration readiness checks and GitSizer assessment
  - Generate migration archives
  - Upload migration archives to the configured storage
  - Start repository migrations
  - Monitor migration progress
  - Run post-migration validation
  - Perform mannequin reclaims (if required)

#### b. GitHub Actions (GHA) Pipeline Migration
- Execute the migration using the provided GitHub Actions workflow.
- The pipeline orchestrates the migration process automatically and provides approvals, reporting, and artifact management.
- The process consists of:
  - Validate runner prerequisites and required tools
  - Validate required variables, secrets, and inventory files
  - Perform migration readiness checks and GitSizer assessment
  - Generate migration archives
  - Upload migration archives to the configured storage
  - Start repository migrations
  - Monitor migration progress
  - Run post-migration validation
  - Generate migration logs, reports, and summary artifacts
  - Perform mannequin reclaims (if required)

## 2. Requirements

### 2.1 Operating System Requirements

| Requirement | Description | Manual GEI Migration | GitHub Actions (GHA) Pipeline Migration |
|------------|-------------|----------------------|------------------------------------------|
| Operating System | Supported operating system used for migration execution. | Ubuntu. | GitHub-hosted runner (`ubuntu-latest`) or a self-hosted runner running Ubuntu. |

### 2.2 Software Requirements

| Requirement | Description | Manual GEI Migration | GitHub Actions (GHA) Pipeline Migration |
|------------|-------------|----------------------|------------------------------------------|
| curl | Used by migration scripts and API operations. | Required. | Validated automatically by the workflow. Installed automatically if missing and the runner has sudo access. |
| jq | Used for JSON processing within migration scripts. | Required. | Validated automatically by the workflow. Installed automatically if missing and the runner has sudo access. |
| git | Required for repository operations and cloning. | Required. | Validated automatically by the workflow. Installed automatically if missing and the runner has sudo access. |
| Docker | Required for building and running the `gl-exporter` container used for migration archive generation. | Required. | Validated automatically by the workflow. Installed automatically if missing and the runner has sudo access. Docker access is also validated. |
| Node.js | Required for JavaScript-based migration utilities. | Node.js v20 or later. | Validated automatically by the workflow. Installed automatically if missing and the runner has sudo access. |
| npm | Required for package dependency management. | npm v10 or later. | Validated automatically by the workflow. Installed automatically if missing and the runner has sudo access. |
| GitHub CLI (`gh`) | Required for inventory generation, migration operations, monitoring, and mannequin management. | Required. | Validated automatically by the workflow. Installed automatically if missing and the runner has sudo access. |

> **Note:** If the GitHub Actions runner user does not have sudo access, the workflow cannot install missing dependencies automatically and will fail with the required remediation actions. If Docker is installed but the runner user does not have permission to execute Docker commands, the runner user must be added to the Docker group.

### 2.3 GitHub Access Requirements

| Requirement | Manual GEI Migration | GitHub Actions (GHA) Pipeline Migration |
|------------|----------------------|------------------------------------------|
| GitHub Access | Access to migration scripts and target GitHub organizations. | Access to migration scripts, workflow execution, workflow artifacts, GitHub environments, approval environments, migration monitoring, and target GitHub organizations. |

### 2.4 Required Token Scopes

#### GitLab API Token

- Must be generated using an administrator account.
- Requires **full API access**.
- Used for:
  - Inventory generation using `gh-gitlab-stats`
  - Migration archive generation using `gl-exporter`

#### GitHub Personal Access Token (PAT)

Required scopes:

- `repo`
- `admin:org`
- `workflow`
- `user`

### 2.5 GitHub CLI extension installation (Manual GEI Migration Only)

The following GitHub CLI extensions are required for Manual GEI Migration. For GitHub Actions (GHA) Pipeline Migration, the workflow automatically installs or upgrades the required extensions.

#### Login to GitHub

##### GitHub Enterprise Cloud
```bash
gh auth login --hostname github.com
```

##### GitHub Enterprise Cloud with Data Residency

```bash
gh auth login --hostname SUBDOMAIN.ghe.com
```

#### Install GitHub CLI Extensions

Install the following GitHub CLI extensions:

**gh-gitlab-stats:** Used to generate GitLab inventory reports.

```bash
gh extension install https://github.com/mona-actions/gh-gitlab-stats
```

**gh-migration-monitor:** Used to monitor migration status manually.

```bash
gh extension install https://github.com/mona-actions/gh-migration-monitor
```

**gh-ado2gh:** Used for:

  - Migration status checks
  - Mannequin CSV generation
  - Mannequin reclamation
  - Migration monitoring

```bash
gh extension install https://github.com/github/gh-ado2gh
```

### 2.6 Intermediate Storage for Migration Archives

Migration archives are temporarily stored before migration into GitHub.

Supported storage options:

| Storage Type | Supported Capacity |
|-------------|-------------------|
| GitHub Storage | Up to 30 GB |
| Azure Storage | Up to 40 GB |
| AWS Storage | Up to 40 GB |

Additional requirements:

- Azure CLI is required when using Azure Storage.
- AWS CLI is required when using AWS Storage.

### 2.7 GitHub Object Storage Feature Flag

The GitHub Object Storage feature flag must be enabled for:

- The GitHub enterprise/account handle.
- All target GitHub organizations.

### 2.8 Network Configuration

The customer is responsible for configuring any required IP allow lists according to their implementation.

Reference documentation:

```text
https://docs.github.com/en/enterprise-cloud/latest/migrations/ado/managing-access-for-a-migration-from-azure-devops#configuring-ip-allow-lists-for-migrations
```

## 3. Migration Package Contents

```text
.
├── .github/workflows/gl-to-gh-migration.yml
├── README.md
├── config.sh
├── runner.sh
├── gl-migration-readiness-check.sh
├── gl-gitsizer-readiness-check.sh
├── generate-gl-migration-archive.sh
├── upload-gl-migration-archive.sh
├── start-gl2gh-repo-migration.sh
├── gl2gh-monitor-migration-status.sh
├── gl-post-migration-validation.sh
├── gitlab-stats-sample.csv
├── gl_exporter/
└── migration_scripts/
    ├── batch.js
    ├── create-env-vars.js
    ├── create-migration-source.js
    ├── gh-api.js
    ├── index.js
    ├── issue.js
    ├── migration.js
    ├── repository.js
    ├── start-repo-migration.js
    ├── state.js
    ├── team.js
    ├── upload-to-github-blob.sh
    ├── upload-to-azure-blob.sh
    ├── upload-to-aws-blob.sh
    ├── user.js
    └── workflow.js
```

## 4. Scripts and Purpose

### 4.1 Shell Scripts

| Script | Purpose |
|------|---------|
| `config.sh` | Contains shared variables used by multiple migration scripts. |
| `runner.sh` | Helper script used to execute GitHub migration operations. |
| `gl-migration-readiness-check.sh` | Checks active merge requests and running pipelines before migration. |
| `gl-gitsizer-readiness-check.sh` | Performs GitSizer analysis to identify repositories and files that may require review before migration. |
| `generate-gl-migration-archive.sh` | Generates GitLab migration archives for repositories defined in the inventory file. |
| `upload-gl-migration-archive.sh` | Uploads generated migration archives to the configured intermediate storage. |
| `start-gl2gh-repo-migration.sh` | Starts GitLab-to-GitHub repository migration jobs. |
| `gl2gh-monitor-migration-status.sh` | Monitors migration progress and generates migration status reports. |
| `gl-post-migration-validation.sh` | Compares branch and commit counts between GitLab and GitHub to validate migration results. |
| `migration_scripts/upload-to-github-blob.sh` | Uploads migration archives to GitHub Storage and generates pre-signed URLs required for repository migration. |
| `migration_scripts/upload-to-azure-blob.sh` | Uploads migration archives to Azure Storage and generates pre-signed URLs required for repository migration. |
| `migration_scripts/upload-to-aws-blob.sh` | Uploads migration archives to AWS S3 Storage and generates pre-signed URLs required for repository migration. |

### 4.2 Scripts in `migration_scripts/` Directory
This directory contains JavaScript modules and helper scripts used to orchestrate GitHub migration operations.

| List of scripts |
|------|
| `batch.js` |
| `create-env-vars.js` |
| `create-migration-source.js` |
| `gh-api.js` |
| `index.js` |
| `issue.js` |
| `migration.js` |
| `repository.js` |
| `start-repo-migration.js` |
| `state.js` |
| `team.js` |
| `user.js` |
| `workflow.js` |

## 5. Pre-Migration

### 5.1 Generate Inventory CSV
Before starting a migration using either the Manual GEI Migration or GitHub Actions (GHA) Pipeline Migration approach, generate an inventory file using the GitHub CLI extension `gitlab-stats`:

```bash
gh gitlab-stats --hostname "gitlab.company.com" --token "glpat-xxxx" --namespace my-gitlab-group
```

This produces a CSV inventory of repositories.

### 5.2 Edit Inventory CSV
After generating the inventory file, update the CSV by adding the following columns:

- `github_org` : Target GitHub Org
- `github_repo` : Target Repo Name
- `gh_repo_visibility` : Supported values: `public, private, internal`

### Optional Export Filters in the Inventory CSV

The inventory CSV supports the following optional columns:

#### `include_in_export`
- Used to export only specific GitLab entities.
- Maps to the `gl-exporter --only` option.
- Can be left empty.

#### `exclude_from_export`
- Used to exclude specific GitLab entities from the export.
- Maps to the `gl-exporter --except` option.
- Can be left empty.

#### Supported Values
The following values are supported:

- merge_requests
- issues
- commit_comments
- hooks
- wiki

#### Multiple Values
Multiple values can be specified using the pipe (`|`) separator. Comma-separated values are not supported for `include_in_export` and `exclude_from_export` columns.

Example:

```csv
include_in_export
issues|merge_requests|commit_comments|hooks|wiki
```
*Note: include_in_export and exclude_from_export are mutually exclusive. If both columns are populated for a repository row, that repository will fail validation and be skipped. The script will continue processing all remaining repositories in the inventory file.*

| Fill in the target GitHub organization and repository name for each row. |

#### Example Inventory CSV

| Namespace | Project | Commit_Count | Branch_Count | Full_URL | github_org | github_repo | gh_repo_visibility | include_in_export | exclude_from_export |
| -------- | -------- | -------- | -------- | -------- | -------- | -------- |-------- | -------- | -------- |
| demo-group/sub-group | demo-project | 20 | 1 | `http://gitlab-server/demo-group/sub-group/demo-project` | ghorg | demoproject | private | merge_requests |
| demo-group-1/sub-group-1 | demo-project-1 | 20 | 1 | `http://gitlab-server/demo-group/sub-group/demo-project-1` | ghorg | demoproject1 | public | | commit_comments |

**Notes**
- The example shows only the minimum required columns.
- The actual inventory CSV may contain additional metadata columns generated by `gh gitlab-stats`.
- Columns `github_org`, `github_repo`, and `gh_repo_visibility` must be populated before running a migration.
- For GitHub Actions (GHA) Pipeline Migration, upload the inventory CSV to the GitHub repository and provide the file name using the `INVENTORY_FILE` workflow input.
- For Manual GEI Migration, export the file name using the `INVENTORY_FILE` environment variable before running the migration scripts.

### 5.3 Upload Inventory to GitHub Repository (GitHub Actions Only)

For GitHub Actions (GHA) Pipeline Migration, upload the updated inventory CSV to the GitHub repository so that the workflow can access it during execution.

This step is not required for Manual GEI Migration if the inventory file is already available on the migration host.

## 6. Manual GEI Migration

The migration can be executed manually by running the provided migration scripts on a supported Ubuntu host.

### 6.1 Export Environment Variables

Export the required environment variables before executing the migration scripts.

| Command | Description |
|----------|-------------|
| `export SOURCE_GL_SERVER_URL="<GitLab server URL>"` | GitLab server URL |
| `export GITLAB_USERNAME="<GitLab username>"` | GitLab username |
| `export GITLAB_API_PRIVATE_TOKEN="<GitLab API token>"` | GitLab API token |
| `export GH_ORG="<GitHub organization>"` | Target GitHub organization |
| `export GH_PAT="<GitHub PAT>"` | GitHub Personal Access Token |
| `export GH_HOST="github.com"` | GitHub Enterprise Cloud host. Use `SUBDOMAIN.ghe.com` for GitHub Enterprise Cloud with Data Residency. |
| `export STORAGE_TYPE="GITHUB"` | Storage type. Supported values: `GITHUB`, `AZURE`, `AWS`. |
| `export INVENTORY_FILE="<inventory-file>.csv"` | Inventory CSV generated using `gh gitlab-stats`. |
| `export AZ_CONTAINER="<container-name>"` | Required only when `STORAGE_TYPE=AZURE`. |
| `export AZURE_STORAGE_CONNECTION_STRING="<connection-string>"` | Required only when `STORAGE_TYPE=AZURE`. |
| `export AWS_BUCKET_NAME="<bucket-name>"` | Required only when `STORAGE_TYPE=AWS`. |
| `export AWS_REGION="<aws-region>"` | Required only when `STORAGE_TYPE=AWS`. |
| `export AWS_ACCESS_KEY_ID="<access-key>"` | Required only when `STORAGE_TYPE=AWS`. |
| `export AWS_SECRET_ACCESS_KEY="<secret-access-key>"` | Required only when `STORAGE_TYPE=AWS`. |

### 6.2 Migration Readiness Check

Run the migration readiness check before starting the migration.

```bash
./gl-migration-readiness-check.sh
```

The script validates:

- Open GitLab merge requests
- Running or pending GitLab pipelines

The script reports any open merge requests and active pipelines found in the selected GitLab projects. Migration can proceed after reviewing the results.

### 6.3 GitSizer Readiness Assessment

Run the GitSizer readiness assessment to identify repositories and files that may require review before migration.

```bash
./gl-gitsizer-readiness-check.sh
```

The assessment generates:

```text
output_files/git-sizer-readiness/
```

The script generates GitSizer reports that can be used to identify repositories and files that may require additional review or planning before migration.

### 6.4 Generate Migration Archives

Generate GitLab migration archives for the repositories defined in the inventory file.

```bash
./generate-gl-migration-archive.sh
```

The script:

- Generates migration archives using `gl-exporter`
- Stores archives in the `gitlab_migration_archives` directory
- Creates an archive inventory CSV file

After successful execution, export the generated archive list file:

```bash
export ARCHIVE_LIST=<path-to-generated-csv>
```

Example:

```bash
export ARCHIVE_LIST=/path/output_files/archive-lists_20250101_103000.csv
```

### 6.5 Upload Migration Archives

Upload migration archives to the configured storage location.

```bash
./upload-gl-migration-archive.sh
```

The script:

- Uploads migration archives to GitHub Storage, Azure Storage, or AWS Storage
- Generates pre-signed URLs for uploaded archives
- Creates a CSV file containing upload details

After successful execution, export the generated upload file:

```bash
export UPLOADED_ARCHIVES=<path-to-presigned-url-csv>
```

Example:

```bash
export UPLOADED_ARCHIVES=/path/output_files/presigned-urls_20250101_110200.csv
```

### 6.6 Start Repository Migrations

Start GitLab-to-GitHub repository migrations.

```bash
./start-gl2gh-repo-migration.sh
```

The script:

- Creates migration sources
- Starts repository migration jobs in GitHub
- Generates migration IDs
- Produces migration output and failure reports

After successful execution, export the migration output file:

```bash
export MIGRATION_OUTPUT_FILE=<path-to-migration-output-csv>
```

Example:

```bash
export MIGRATION_OUTPUT_FILE=/path/output_files/migration-outputs_20250101_112000.csv
```

### 6.7 Monitor Repository Migrations

Monitor migration progress until all repository migrations are completed.

```bash
./gl2gh-monitor-migration-status.sh
```

The script:

- Reads migration IDs from the migration output file
- Monitors migration progress
- Generates migration status reports

Output:

```text
migration-status-<timestamp>.csv
```

### 6.8 Post-Migration Validation

Validate migrated repositories after migration completion.

```bash
./gl-post-migration-validation.sh
```

The validation process compares GitLab and GitHub repository metadata, including:

- Repository existence
- Branch counts
- Commit counts

Generated outputs include:

```text
validation-summary.csv
validation-summary.md
```

### 6.9 Perform Mannequin Reclaims (Optional)

If GitLab users are migrated as mannequins, perform user identity mapping and mannequin reclamation as described in Section 8.


## 7. GitHub Actions (GHA) Pipeline Migration

### 7.1 GitHub Environment Setup

The workflow uses GitHub Environments to store migration configuration, secrets, and approval controls.

#### CI/CD Environment

Create a GitHub Environment containing the variables and secrets required for migration.

Navigate to:

```text
GitHub Repository → Settings → Environments → <ENVIRONMENT_NAME>
```

Example:

```text
customer-prod-env
```

The environment name is provided as an input when executing the workflow.

The following workflow jobs use this environment:

- `getting-env-ready`
- `validate-prerequisites`
- `pre-migration-readiness-check`
- `generate-migration-archives`
- `upload-migration-archives`
- `start-repository-migration`
- `display-migration-summary`
- `monitor-repository-migrations`
- `post-migration-validation`

##### Environment Variables

| Name | Description |
|--------|-------------|
| `SOURCE_GL_SERVER_URL` | GitLab server URL |
| `GITLAB_USERNAME` | GitLab username |
| `GH_HOST` | `github.com` or `SUBDOMAIN.ghe.com` |
| `STORAGE_TYPE` | `GITHUB`, `AZURE`, or `AWS` |
| `AZ_CONTAINER` | Required when `STORAGE_TYPE=AZURE` |
| `AWS_BUCKET_NAME` | Required when `STORAGE_TYPE=AWS` |
| `AWS_REGION` | Required when `STORAGE_TYPE=AWS` |

##### Environment Secrets

| Name | Description |
|--------|-------------|
| `GITLAB_API_PRIVATE_TOKEN` | GitLab API token |
| `GH_PAT` | GitHub Personal Access Token |
| `AZURE_STORAGE_CONNECTION_STRING` | Required when `STORAGE_TYPE=AZURE` |
| `AWS_ACCESS_KEY_ID` | Required when `STORAGE_TYPE=AWS` |
| `AWS_SECRET_ACCESS_KEY` | Required when `STORAGE_TYPE=AWS` |

#### Approval Environment

Create a separate GitHub Environment for approval workflows.

Navigate to:

```text
GitHub Repository → Settings → Environments 
```
Create environemnt called `approvers-group`

This environment is used by workflow approval stages, including:

- Approval after readiness checks
- Approval before migration monitoring

Configure reviewers as required by your organization's approval process.

### 7.2 Pipeline Flow

The GitHub Actions workflow automates the migration process using the following stages:

#### 1. Getting Environment Ready

- Validates the runner operating system
- Validates required tools and dependencies
- Validates Docker access
- Installs missing dependencies (when permitted)
- Authenticates GitHub CLI
- Installs required GitHub CLI extensions

#### 2. Validate Prerequisites

- Validates required variables and secrets
- Validates storage configuration
- Validates workflow inputs
- Validates required scripts
- Validates inventory file configuration

#### 3. Pre-Migration Readiness Check

- Runs migration readiness checks
- Executes GitSizer analysis
- Generates readiness reports
- Uploads readiness reports as workflow artifacts

#### 4. Approval Stage

- Uses the configured approval environment
- Allows readiness results to be reviewed before continuing

#### 5. Generate Migration Archives

- Builds the `gl-exporter` Docker image if required
- Generates migration archives
- Creates archive inventory files
- Uploads logs and outputs as workflow artifacts

#### 6. Upload Migration Archives

- Uploads archives to the configured storage platform
- Generates upload inventory files
- Uploads logs and outputs as workflow artifacts

#### 7. Start Repository Migrations

- Creates migration sources
- Starts GitLab-to-GitHub repository migrations
- Generates migration output files
- Uploads logs and outputs as workflow artifacts

#### 8. Display Migration Summary

- Aggregates migration results
- Displays migration statistics
- Generates a final migration summary report

#### 9. Approval Before Monitoring

- Uses the configured approval environment
- Allows migration results to be reviewed before monitoring begins

#### 10. Monitor Repository Migrations

- Authenticates GitHub CLI
- Installs or upgrades `gh-ado2gh`
- Determines the appropriate GitHub API endpoint
- Monitors repository migration status
- Generates migration status reports

#### 11. Post-Migration Validation

- Validates successfully migrated repositories
- Executes branch and commit validation checks
- Generates validation reports

#### 12. Artifact Preservation

The workflow uploads migration artifacts including:

- Readiness reports
- Archive generation outputs
- Archive upload outputs
- Migration outputs
- Migration summaries
- Migration status reports
- Validation reports
- Workflow logs

### 7.3 Executing the Pipeline

1. Open the GitHub repository.
2. Navigate to:

```text
Actions → GitLab to GitHub Migration Pipeline
```

3. Select **Run workflow**.
4. Provide the required inputs:

| Input | Description |
|---------|-------------|
| Environment Name | GitHub Environment containing migration variables and secrets |
| Inventory File | Inventory CSV generated using `gh gitlab-stats` |
| GitHub Type | `GitHub` or `GitHubDR` |
| Runner Label | GitHub-hosted or self-hosted runner label |

5. Select **Run workflow** to start the migration.

### 7.4 Artifacts and Retention

The workflow uploads artifacts to support troubleshooting and migration validation.

Typical artifacts include:

- Readiness reports
- GitSizer reports
- Archive generation outputs
- Archive upload outputs
- Migration output files
- Migration summaries
- Migration status reports
- Validation reports
- Workflow logs

Artifact retention is controlled through the workflow configuration.

## 8. User Identity Mapping (Mannequins)

During migration, GitLab users that cannot be automatically mapped to GitHub users are imported as **mannequins**. Mannequin reclamation allows these placeholder identities to be associated with real GitHub users after migration.

### 8.1 Generate Mannequin Mapping File

Generate a CSV file containing mannequin users for a GitHub organization.

#### GitHub Enterprise Cloud

```bash
gh ado2gh generate-mannequin-csv --github-org "{github-org}"
```

#### GitHub Enterprise Cloud with Data Residency

```bash
gh ado2gh generate-mannequin-csv \
  --github-org "{github-org}" \
  --target-api-url https://api.SUBDOMAIN.ghe.com
```

The command generates a `mannequins.csv` file containing the mannequin identities detected within the target GitHub organization.

### 8.2 Update Mannequin Mapping

Open `mannequins.csv` and populate the **target-user** column with valid GitHub usernames.

#### Example

| mannequin-user | mannequin-id | target-user |
|----------------|--------------|-------------|
| gluser1 | M_kgDODtfbRA | github-user1 |
| gluser2 | M_kgDODtfbRg | github-user2 |

**Notes**

- During migration, GitLab users that cannot be automatically matched to GitHub users are imported as mannequins.
- Update the `target-user` column with the appropriate GitHub username for each mannequin account.
- The referenced GitHub users must exist within the target GitHub organization.
- Mannequin reclamation updates migrated content such as commits, issues, comments, and pull requests to reference the mapped GitHub user.

### 8.3 Reclaim Mannequins

After updating the mapping file, execute the mannequin reclamation command.

#### GitHub Enterprise Cloud

```bash
gh ado2gh reclaim-mannequin \
  --github-org "{github-org}" \
  --csv mannequins.csv \
  --skip-invitation
```

#### GitHub Enterprise Cloud with Data Residency

```bash
gh ado2gh reclaim-mannequin \
  --github-org "{github-org}" \
  --csv mannequins.csv \
  --skip-invitation \
  --target-api-url https://api.SUBDOMAIN.ghe.com
```

## 9. Appendix

### 9.1 Install GitHub CLI

Install GitHub CLI by following the official installation documentation:

```text
https://github.com/cli/cli#installation
```

### 9.2 Install GitHub CLI Extensions

For GitHub Actions (GHA) Pipeline Migration, the workflow automatically installs or upgrades the required GitHub CLI extensions during the `getting-env-ready` stage.

For Manual GEI Migration, install the following extensions manually.

#### gh-gitlab-stats

Used to generate GitLab inventory reports.

```bash
gh extension install https://github.com/mona-actions/gh-gitlab-stats
```

#### gh-migration-monitor

Used to monitor repository migration status.

```bash
gh extension install https://github.com/mona-actions/gh-migration-monitor
```

#### gh-ado2gh

Used for:

- Migration status checks
- Mannequin CSV generation
- Mannequin reclamation
- Repository migration monitoring

```bash
gh extension install https://github.com/github/gh-ado2gh
```

### 9.3 Build gl-exporter Docker Image

For GitHub Actions (GHA) Pipeline Migration, the workflow automatically builds the `gl-exporter` Docker image if it is not already available.

For Manual GEI Migration, build the image using the following commands:

```bash
cd gl_exporter
docker build --no-cache=true -t gl-exporter .
```

Verify the image:

```bash
docker images | grep "gl-exporter"
```

Example output:

```text
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
gl-exporter   latest    5e168437a7a1   12 hours ago   1.51GB
ruby          3.2.1     3440a912810a   2 years ago    893MB
```

### 9.4 Check Migration Status by Migration ID

#### GitHub Enterprise Cloud

```bash
gh ado2gh wait-for-migration --migration-id <migration-id>
```

#### GitHub Enterprise Cloud with Data Residency

```bash
gh ado2gh wait-for-migration \
  --migration-id <migration-id> \
  --target-api-url https://api.SUBDOMAIN.ghe.com
```

### 9.5 Monitor Migrations Using gh-migration-monitor

#### GitHub Enterprise Cloud

```bash
gh migration-monitor --organization <github-org> --github-token <gh-pat>
```

#### GitHub Enterprise Cloud with Data Residency

```text
gh-migration-monitor is not supported for GitHub Enterprise Cloud with Data Residency.
```
