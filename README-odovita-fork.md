# OpenEMR Enhanced Fork

This is our enhanced fork of the [OpenEMR](https://github.com/openemr/openemr) electronic health records system. We maintain this fork to accelerate development and deployment of critical fixes and features while continuing to contribute back to the upstream project.

## Why This Fork Exists

The upstream OpenEMR project has a slow integration process for pull requests. While this ensures stability, it creates delays when we need urgent fixes or features for our production environment. This fork allows us to:

-   **Deploy fixes immediately** without waiting for upstream approval
-   **Test new features** in our environment before they're accepted upstream
-   **Maintain stability** by controlling what gets deployed to production
-   **Continue contributing** to the open source project with clean PRs

## What's Different

This fork includes:

-   Bug fixes that haven't been merged upstream yet
-   Performance improvements specific to our use case
-   Custom features that may eventually be contributed back
-   Experimental changes being tested before upstream submission

## Installation

Install directly from this repository instead of the official OpenEMR:

```bash
git clone https://github.com/[your-username]/openemr.git
cd openemr
git checkout production-rel-8_5_0
# Follow standard OpenEMR installation process
```

## Contributing

We welcome contributions! Please follow our branching strategy below to ensure clean integration.

## Branching Strategy

### Main Branches

-   **`upstream-sync-<version>`** - Clean mirror of official OpenEMR version branch (e.g., `upstream-sync-8_0_0`)
-   **`production-rel-<version>`** - Our deployment branch for a given upstream version (e.g., `production-rel-8_5_0`)

### The Process

We maintain our own fast-moving production environment while syncing with stable upstream release branches (NOT master, which contains unreleased code).

### Initial Setup

```bash
# Add upstream remote (one time setup)
git remote add upstream https://github.com/openemr/openemr.git

# Fetch all upstream branches and tags
git fetch upstream --tags
```

### Syncing with a New Upstream Release

**IMPORTANT**: Never sync with upstream's `master` branch — it contains unreleased code. Always sync with official release tags or release branches.

Preferred: pin to a specific release tag (e.g., `v8_5_0`) for reproducibility.

```bash
# 1. Determine the latest stable release version
# Check https://github.com/openemr/openemr/releases for the latest tag

# 2. Create a new production branch from the upstream release tag
git fetch upstream --tags
git checkout -b production-rel-8_5_0 v8_5_0

# 3. Cherry-pick our custom commits from the previous production branch
#    in chronological order. Use:
git log --reverse --format="%h %ai %s" v8_4_1..production-rel-8_4_1
#    then cherry-pick each commit (skip any that have since been merged
#    upstream — see "Custom Commits Overview" below). The dispense
#    backport is regenerated from the PR branch instead of cherry-picked.

# 4. Push the new branch
git push -u origin production-rel-8_5_0
```

Before cherry-picking, check `git log v8_<old>..upstream/master` for post-release fixes to areas the clinic uses (documents, prescriptions, patient forms). A `.1` patch release can ship a regression whose fix only landed on master afterwards (8.4.1 did: document delete, upstream #14262).

### Creating a New Feature/Fix

```bash
# 1. Start from the current production branch
git checkout production-rel-8_5_0

# 2. Create feature branch
git checkout -b feature/your-feature-name

# 3. Make your changes and commit
git add .
git commit -m "Fix: your changes"

# 4. Push feature branch
git push -u origin feature/your-feature-name

# 5. Merge into the production branch when ready
git checkout production-rel-8_5_0
git merge feature/your-feature-name
git push origin production-rel-8_5_0
```

### Creating PR to Upstream (Optional)

Only create PRs for features that would benefit the broader OpenEMR community:

1. Go to GitHub and create PR from your `feature/your-feature-name` branch
2. Target the upstream repository's appropriate release branch
3. **Never create PRs from your production branch**
4. Not every feature needs to be contributed — keep organization-specific customizations in your fork

### Important Notes

-   **Never sync with upstream's `master` branch** — it contains unreleased, unstable code
-   **Always sync with versioned release tags** (e.g., `v8_5_0`) for reproducibility
-   **Never use GitHub's "Sync fork" button** — it creates messy merge commits
-   Always create feature branches from the current `production-rel-<version>` branch for clean history
-   Deploy manually from the appropriate `production-rel-<version>` branch when changes are stable

## Build Production Image From odovita/openemr

Since 8.1 the production image is built from `docker/release/` inside this repository; the old `docker/openemr/<version>` contexts in `openemr-devops` were deleted upstream in June 2026 (the local `odovita-deployment` branch of openemr-devops still carries the 8.0.0.3 context used for the last 8.0.0.3 images). The Dockerfile clones the source from a repository and ref given as build arguments, so no copy of the context is needed any more.

1. Check out the production branch of this fork and build from its `docker/release` context. `OPENEMR_GIT` and `OPENEMR_VERSION` select what gets cloned into the image (both are upstream build arguments since 8.5.0; the 8.4.1 fork carried its own `OPENEMR_REPO` for the same purpose). The branch must already be pushed, because the Dockerfile clones it from GitHub:

    ```bash
    git checkout production-rel-8_5_0
    docker buildx build \
      --platform linux/amd64 \
      --build-arg OPENEMR_GIT=https://github.com/odovita/openemr.git \
      --build-arg OPENEMR_VERSION=production-rel-8_5_0 \
      -t ghcr.io/odovita/openemr:production-8.5.0 \
      docker/release
    ```

    `PHP_VERSION` and `ALPINE_VERSION` default to what upstream ships for this release; override them only for security pins (as was done for php2shell in October 2026).

2. (Optional) Smoke-test the image with the compose file in `docker/production/` before pushing.

3. Push to the registry (`--push`) when you are ready, then roll out on the Lightsail host (`~/lightsail`, `sudo docker-compose pull && sudo docker-compose up -d`). The Lightsail compose runs MySQL 8.4 with `--mysql-native-password=ON --tls-version=` and bind-mounts a `[client] skip-ssl` file as `/root/.my.cnf` into the openemr container, because the MariaDB 11.8 client in the release image requires TLS and cannot verify MySQL's self-signed certificate. Keep both when editing the compose file.

4. The release image upgrades the database automatically on the first start of a new version (the entrypoint compares its docker-version marker and runs `sql_upgrade.php`; watch the container log for "Upgrade detected"). Apply `sql/patient-to-patient-relationships.sql` if the site does not have the `patient_relationships` table yet (see "Custom Commits Overview").

### Keeping The Fork Updated

When a new OpenEMR version is released, follow the "Syncing with a New Upstream Release" steps above, then rebuild your Docker image with the new version number.

## License

Same as upstream OpenEMR - GNU General Public License v3.0

---

## Technical Notes for Claude

This section contains technical context for AI assistants working on this codebase.

### Custom Commits Overview

When rebasing/cherry-picking our custom commits onto a new upstream version, be aware that some fixes have been incorporated upstream:

#### Drug Dispensing Fix (PR #8598)
- **Original issue**: Drug dispensing failed silently when no encounter was selected
- **Our fix**: Added encounter validation in `sellDrug()` function with user-friendly error messages
- **Status as of rel-704 / 8.0.0**: **MERGED UPSTREAM** — incorporated into `src/Services/DrugSalesService.php` (see the encounter sanity check around lines 243-258) with explicit credit to PR #8598 in a code comment
- **Action**: Do not cherry-pick `387709116` (Improve drug sale error message) or `a28f71c5a` (Render actual drugId as option value) when syncing to rel-704 or later — the fixes are part of the official codebase

#### Patient-to-Patient Relationships Feature
- **Description**: Custom feature allowing linking patients to each other (e.g., family relationships)
- **Commits on `production-rel-8_5_0`**: `5633ffb7e3` + `b26fa04a17` (tests) + `f99bbb03df` (fork-specific tweaks) + `4578e7bf30` (BS5 modal fix) + `1567890632` (8.4.1 API adaptation: `CsrfUtils` needs the session, the table references `patient_data.id` rather than `pid`; still required on 8.5.0)
- **Upstream PR**: https://github.com/openemr/openemr/pull/10483, open and unreviewed as of October 2026; the feature is not in 8.5.0
- **Schema**: `sql/patient-to-patient-relationships.sql` is NOT applied by the regular upgrade. Run it once per site (fresh install or new database) from the web root as the web-server user (8.4 refuses the CLI as root), e.g. inside the container:
  `docker compose exec -u apache openemr sh -c 'cd /var/www/localhost/htdocs/openemr && php -r '"'"'$_GET["site"]="default"; $ignoreAuth=true; require_once "interface/globals.php"; require_once "library/sql_upgrade_fx.php"; upgradeFromSqlFile("patient-to-patient-relationships.sql");'"'"''`
  The file uses the `#IfNotTable` guards, so running it twice is harmless.
- **Files added**:
  - `src/Services/PatientRelationshipService.php`
  - `src/Entity/PatientRelationship.php`
  - `src/Patient/Cards/PatientRelationshipViewCard.php`
  - `interface/patient_file/summary/patient_relationships_ajax.php`
  - `templates/patient/card/patient_relationships.html.twig`
  - `sql/patient-to-patient-relationships.sql`
  - `tests/Tests/Entity/PatientRelationshipTest.php`
  - `tests/Tests/Services/PatientRelationshipServiceTest.php`
- **Files modified**:
  - `interface/patient_file/summary/demographics.php` — added card registration
  - `src/Common/Uuid/UuidRegistry.php` — added `patient_relationships` table entry
- **Status**: Custom feature, not submitted upstream

#### Patient Photo Webcam Capture
- **Description**: Capture patient photos via webcam directly from the new-patient form and the demographics view; click-to-view existing photos
- **Commits on `production-rel-8_5_0`**: `a629cc1259` + `b14c84d077` + `c721786fb2` (SystemLogger import fix) + `5cf53eb0f9` (button styling) + `4d02096912` (`?v=` cache buster on the script include, required by the 8.5.0 `AssetCacheBusterTest`); the `save_patient_photo.php` part of `1567890632` belongs to this feature as well
- **Upstream PR**: https://github.com/openemr/openemr/pull/10157, open and unreviewed as of October 2026; the feature is not in 8.5.0
- **Sync note**: all commits applied onto 8.5.0 without conflicts, including `interface/new/new_comprehensive.php` and `new_comprehensive_save.php`, which upstream keeps changing; re-check those two files first on every sync
- **Files added**:
  - `library/js/webcam-capture.js`
  - `library/ajax/save_patient_photo.php`
  - `src/Services/PatientPhotoService.php`
- **Files modified**:
  - `interface/main/tabs/js/application_view_model.js` — patient picture click handler + photo dialog
  - `interface/main/tabs/main.php`, `interface/main/tabs/templates/patient_data_template.php`
  - `interface/new/new_comprehensive.php`, `interface/new/new_comprehensive_save.php`
  - `interface/patient_file/summary/demographics.php`
  - `interface/themes/tabs_style_compact.scss`, `interface/themes/tabs_style_full.scss`
  - `src/Services/PatientService.php`
- **Status**: Custom feature, not submitted upstream

#### Docker Development Fixes
- **Description**: Volume tweak in `docker-compose.yml` to prevent permission/rsync errors during development
- **Files modified**:
  - `docker/development-easy-light/docker-compose.yml`
- **Status**: Development convenience, may or may not be needed depending on upstream changes
- **Note**: This change rides along inside `f99bbb03df` rather than as a standalone commit

#### Dispense Flow Refactor + Lot Picker (upstream PR #12098)
- **Description**: Unified "Save and Dispense" flow, lot picker with the `restrict_sales_to_default_warehouse` global, sales/inventory reversal on prescription delete, bottle-label page `interface/drugs/print_drug_label.php`, `drugs.inc.php` reduced to `interface/drugs/inventory_acl.inc.php`
- **Commit on `production-rel-8_5_0`**: `79a68d5ec2` — a squashed backport of https://github.com/openemr/openemr/pull/12098 (branch `refactor/unify-dispense-flow-upstream`, head `d4a0c38f58` at the time). The PR's master base `fd53b1395a` is contained in `v8_5_0`, so the squash is byte-identical to the PR diff and carries no 8.5.0-specific adaptation.
- **Status**: Open upstream PR, under review. On the next sync check whether it has been merged into the target release; if so, drop this commit instead of cherry-picking it. If it is still open, regenerate the squash from the PR branch (`git diff <merge-base> <pr-branch> | git apply --3way --index`) rather than cherry-picking the old squash.
- **New global**: "Restrict Drug Sales to Default Warehouse" (Features), default on. It restricts users who have a default warehouse to that warehouse; a user without one is not restricted (legacy behaviour), and the dispense card shows a status line saying which case applies. Make sure every dispensing user at Odovita has a default warehouse set under Administration → Users.

#### Document Delete Fix (upstream #14262)
- **Description**: 8.4.1 shipped upstream #13958 (document deletion requires patient context) without the follow-up fix #14262, so the Delete button on the document view page answered "Access denied"
- **Status**: **MERGED UPSTREAM**, contained in 8.5.0. `production-rel-8_4_1` carried it as a cherry-pick (`e28aa26774`); do not carry that commit forward.

### Branch Management

- **Backup branches**: `production-rel-7_0_3_4-squashed` (4 squashed commits from the 7.0.3 era), `production-rel-7_0_4_1` (8 custom commits on top of rel-704), `production-rel-8_0_0_3` (the 8.0.0.3 base), `feat/dispense-lot-picker-prod-8003` (8.0.0.3 + the May 2026 dispense work, deployed until the 8.4.1 rollout), `production-rel-8_4_1` (deployed at Odovita from 2026-10-23 until the 8.5.0 rollout; includes the document delete cherry-pick that never shipped as an image)
- **Production branches**: Named `production-rel-X_Y_Z` (e.g., `production-rel-8_5_0`)
- Always check if fixes have been merged upstream before cherry-picking
