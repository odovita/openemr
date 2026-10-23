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
git checkout production-rel-8_4_1
# Follow standard OpenEMR installation process
```

## Contributing

We welcome contributions! Please follow our branching strategy below to ensure clean integration.

## Branching Strategy

### Main Branches

-   **`upstream-sync-<version>`** - Clean mirror of official OpenEMR version branch (e.g., `upstream-sync-8_0_0`)
-   **`production-rel-<version>`** - Our deployment branch for a given upstream version (e.g., `production-rel-8_4_1`)

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

Preferred: pin to a specific release tag (e.g., `v8_4_1`) for reproducibility.

```bash
# 1. Determine the latest stable release version
# Check https://github.com/openemr/openemr/releases for the latest tag

# 2. Create a new production branch from the upstream release tag
git fetch upstream --tags
git checkout -b production-rel-8_4_1 v8_4_1

# 3. Cherry-pick our custom commits from the previous production branch
#    in chronological order. Use:
git log --reverse --format="%h %ai %s" v8_0_0_3..production-rel-8_0_0_3
#    then cherry-pick each commit (skip any that have since been merged
#    upstream — see "Custom Commits Overview" below).

# 4. Push the new branch
git push -u origin production-rel-8_4_1
```

### Creating a New Feature/Fix

```bash
# 1. Start from the current production branch
git checkout production-rel-8_4_1

# 2. Create feature branch
git checkout -b feature/your-feature-name

# 3. Make your changes and commit
git add .
git commit -m "Fix: your changes"

# 4. Push feature branch
git push -u origin feature/your-feature-name

# 5. Merge into the production branch when ready
git checkout production-rel-8_4_1
git merge feature/your-feature-name
git push origin production-rel-8_4_1
```

### Creating PR to Upstream (Optional)

Only create PRs for features that would benefit the broader OpenEMR community:

1. Go to GitHub and create PR from your `feature/your-feature-name` branch
2. Target the upstream repository's appropriate release branch
3. **Never create PRs from your production branch**
4. Not every feature needs to be contributed — keep organization-specific customizations in your fork

### Important Notes

-   **Never sync with upstream's `master` branch** — it contains unreleased, unstable code
-   **Always sync with versioned release tags** (e.g., `v8_4_1`) for reproducibility
-   **Never use GitHub's "Sync fork" button** — it creates messy merge commits
-   Always create feature branches from the current `production-rel-<version>` branch for clean history
-   Deploy manually from the appropriate `production-rel-<version>` branch when changes are stable

## Build Production Image From odovita/openemr

Since 8.1 the production image is built from `docker/release/` inside this repository; the old `docker/openemr/<version>` contexts in `openemr-devops` were deleted upstream in June 2026 (the local `odovita-deployment` branch of openemr-devops still carries the 8.0.0.3 context used for the last 8.0.0.3 images). The Dockerfile clones the source from a repository and ref given as build arguments, so no copy of the context is needed any more.

1. Check out the production branch of this fork and build from its `docker/release` context. `OPENEMR_REPO` and `OPENEMR_VERSION` select what gets cloned into the image (the fork adds `OPENEMR_REPO`; upstream only has `OPENEMR_VERSION`):

    ```bash
    git checkout production-rel-8_4_1
    docker buildx build \
      --platform linux/amd64 \
      --build-arg OPENEMR_REPO=https://github.com/odovita/openemr.git \
      --build-arg OPENEMR_VERSION=production-rel-8_4_1 \
      -t ghcr.io/odovita/openemr:production-8.4.1 \
      docker/release
    ```

    `PHP_VERSION` and `ALPINE_VERSION` default to what upstream ships for this release; override them only for security pins (as was done for php2shell in October 2026).

2. (Optional) Smoke-test the image with the compose file in `docker/production/` before pushing.

3. Push to the registry (`--push`) when you are ready, then roll out on the Lightsail host (`~/lightsail`, `sudo docker-compose pull && sudo docker-compose up -d`).

4. Run the database upgrade once after the first start on the new version (Administration → Upgrade, or `sql_upgrade.php`), and apply `sql/patient-to-patient-relationships.sql` if the site does not have the `patient_relationships` table yet (see "Custom Commits Overview").

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
- **Commits on `production-rel-8_4_1`**: `8822946190` + `028d2234b6` (tests) + `44d8bd5bf7` (fork-specific tweaks) + `ffca13bdd0` (BS5 modal fix)
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
- **Commits on `production-rel-8_4_1`**: `8ce94467c2` + `db0d49d3f8` + `c7c5425458` (SystemLogger import fix, a no-op on 8.4.1) + `1b35c0bfbf` (button styling)
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
- **Note**: This change rides along inside `44d8bd5bf7` rather than as a standalone commit

#### Dispense Flow Refactor + Lot Picker (upstream PR #12098)
- **Description**: Unified "Save and Dispense" flow, lot picker with the `restrict_sales_to_default_warehouse` global, sales/inventory reversal on prescription delete, bottle-label page `interface/drugs/print_drug_label.php`, `drugs.inc.php` reduced to `interface/drugs/inventory_acl.inc.php`
- **Commit on `production-rel-8_4_1`**: `da525d3dca` — a squashed backport of https://github.com/openemr/openemr/pull/12098 (22 commits on master, head `c4461b32fe` at the time)
- **Status**: Open upstream PR. On the next sync check whether it has been merged into the target release; if so, drop this commit instead of cherry-picking it. If it is still open, regenerate the squash from the PR branch (`git diff <merge-base> <pr-branch> | git apply -3`) rather than re-resolving the 8.4.1 conflicts by hand.
- **Differences to master**: `DrugSalesService::send_drug_email()` keeps PHPMailer (MyMailer only exists after 8.4.1); the PHPStan baseline changes of the PR are not carried.
- **New global**: "Restrict Drug Sales to Default Warehouse" (Features), default on. It restricts users who have a default warehouse to that warehouse; a user without one is not restricted (legacy behaviour), and the dispense card shows a status line saying which case applies. Make sure every dispensing user at Odovita has a default warehouse set under Administration → Users.

### Branch Management

- **Backup branches**: `production-rel-7_0_3_4-squashed` (4 squashed commits from the 7.0.3 era), `production-rel-7_0_4_1` (8 custom commits on top of rel-704), `production-rel-8_0_0_3` (the 8.0.0.3 base), `feat/dispense-lot-picker-prod-8003` (8.0.0.3 + the May 2026 dispense work, deployed until the 8.4.1 rollout)
- **Production branches**: Named `production-rel-X_Y_Z` (e.g., `production-rel-8_4_1`)
- Always check if fixes have been merged upstream before cherry-picking
