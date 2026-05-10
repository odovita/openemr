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
git checkout production-rel-8_0_0_3
# Follow standard OpenEMR installation process
```

## Contributing

We welcome contributions! Please follow our branching strategy below to ensure clean integration.

## Branching Strategy

### Main Branches

-   **`upstream-sync-<version>`** - Clean mirror of official OpenEMR version branch (e.g., `upstream-sync-8_0_0`)
-   **`production-rel-<version>`** - Our deployment branch for a given upstream version (e.g., `production-rel-8_0_0_3`)

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

Preferred: pin to a specific release tag (e.g., `v8_0_0_3`) for reproducibility.

```bash
# 1. Determine the latest stable release version
# Check https://github.com/openemr/openemr/releases for the latest tag

# 2. Create a new production branch from the upstream release tag
git fetch upstream --tags
git checkout -b production-rel-8_0_0_3 v8_0_0_3

# 3. Cherry-pick our custom commits from the previous production branch
#    in chronological order. Use:
git log --reverse --format="%h %ai %s" upstream/rel-704..production-rel-7_0_4_1
#    then cherry-pick each commit (skip any that have since been merged
#    upstream — see "Custom Commits Overview" below).

# 4. Push the new branch
git push -u origin production-rel-8_0_0_3
```

### Creating a New Feature/Fix

```bash
# 1. Start from the current production branch
git checkout production-rel-8_0_0_3

# 2. Create feature branch
git checkout -b feature/your-feature-name

# 3. Make your changes and commit
git add .
git commit -m "Fix: your changes"

# 4. Push feature branch
git push -u origin feature/your-feature-name

# 5. Merge into the production branch when ready
git checkout production-rel-8_0_0_3
git merge feature/your-feature-name
git push origin production-rel-8_0_0_3
```

### Creating PR to Upstream (Optional)

Only create PRs for features that would benefit the broader OpenEMR community:

1. Go to GitHub and create PR from your `feature/your-feature-name` branch
2. Target the upstream repository's appropriate release branch
3. **Never create PRs from your production branch**
4. Not every feature needs to be contributed — keep organization-specific customizations in your fork

### Important Notes

-   **Never sync with upstream's `master` branch** — it contains unreleased, unstable code
-   **Always sync with versioned release tags** (e.g., `v8_0_0_3`) for reproducibility
-   **Never use GitHub's "Sync fork" button** — it creates messy merge commits
-   Always create feature branches from the current `production-rel-<version>` branch for clean history
-   Deploy manually from the appropriate `production-rel-<version>` branch when changes are stable

## Build Production Image From odovita/openemr

The steps below live in the `openemr-devops` tooling repo but build the application from this fork.

1. Clone the tooling repo and enter it:

    ```bash
    git clone https://github.com/openemr/openemr-devops.git
    cd openemr-devops
    ```

2. Find the correct version directory (e.g., 8.0.0) and copy it:

    ```bash
    cp -a docker/openemr/8.0.0 docker/openemr/8.0.0-odovita
    ```

3. Edit `docker/openemr/8.0.0-odovita/Dockerfile` and change the `git clone` line to pull from our fork's production branch:

    ```Dockerfile
    RUN apk add --no-cache build-base \
        && git clone --branch production-rel-8_0_0_3 https://github.com/odovita/openemr.git --depth 1 \
        && rm -rf openemr/.git \
        && cd openemr \
        ...
    ```

    Keep the rest of the build block unchanged so Composer/NPM steps remain identical to the official image.

4. Build the Docker image through the bundled `docker compose` stack:

    ```bash
    cd docker/openemr
    export DOCKER_CONTEXT_PATH=8.0.0-odovita
    COMPOSE_PROFILES=prod docker compose build
    ```

5. (Optional) Smoke-test the container with MariaDB:

    ```bash
    COMPOSE_PROFILES=prod docker compose up -d --wait
    # browse http://localhost:8080 or inspect logs
    docker compose logs openemr
    docker compose down --volumes
    ```

6. Push a multi-arch image to your registry when you are ready:

    ```bash
    docker buildx build \
      --platform linux/amd64,linux/arm64 \
      -t ghcr.io/odovita/openemr:8.0.0-odovita \
      --push docker/openemr/8.0.0-odovita
    ```

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
- **Commits**: `e50889c42` + `108693d6c` (tests) + `7902b5248` (fork-specific tweaks) + `77c0d9841` (BS5 modal fix)
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
- **Commits**: `4fc7019be` + `fdd8f0dd8` + `57846d98e` (SystemLogger import fix) + `7577d73ec` (button styling)
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
- **Note**: This change rides along inside `7902b5248` rather than as a standalone commit

### Branch Management

- **Backup branches**: `production-rel-7_0_3_4-squashed` (4 squashed commits from the 7.0.3 era), `production-rel-7_0_4_1` (8 custom commits on top of rel-704)
- **Production branches**: Named `production-rel-X_Y_Z` (e.g., `production-rel-8_0_0_3`)
- Always check if fixes have been merged upstream before cherry-picking
