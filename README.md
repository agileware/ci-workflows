# CI Workflows

Reusable GitHub Actions workflows for Agileware's CiviCRM extension and WordPress plugin
repos. Most workflows spin up a WordPress + CiviCRM environment in Docker and run a specific
task against it; the release workflows operate on git/GitHub directly instead.

- [civix Upgrade Workflow](#civix-upgrade-workflow) — runs `civix upgrade` and opens a PR with the result
- [CiviCRM PHPUnit Testing Workflow](#civicrm-phpunit-testing-workflow) — runs an extension's headless PHPUnit suite
- [Playwright Frontend Testing Workflow](#playwright-frontend-testing-workflow) — runs an extension's Playwright test suite
- [WordPress Plugin Cut Release Workflow](#wordpress-plugin-cut-release-workflow) — builds a filtered release branch, tags it, and publishes the GitHub Release
- [CiviCRM Extension Cut Release Workflow](#civicrm-extension-cut-release-workflow) — same, versioned from info.xml instead of a plugin header

## civix Upgrade Workflow

This reusable GitHub Actions workflow sets up a WordPress + CiviCRM environment and runs `civix upgrade` on a CiviCRM extension, then opens a pull request with the regenerated files.

### 📂 File Location

```
.github/workflows/civix-upgrade.yml
```

### 🔧 Usage

To use this workflow in another repo, add a workflow such as `.github/workflows/upgrade.yml`:

```yaml
name: Upgrade Extension

on:
  push

jobs:
  upgrade:
    uses: agileware/ci-workflows/.github/workflows/civix-upgrade.yml@main
    secrets:
      DOCKERHUB_USER: ${{ secrets.DOCKERHUB_USER }}
      DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
    # Optional
    # with:
    #   EXTENSION_NAME: my_custom_extension
```

### 🔐 Required Secrets

- `DOCKERHUB_USER`
- `DOCKERHUB_TOKEN`

### 🧠 Notes

- If `EXTENSION_NAME` is not provided, the workflow defaults to the name of the calling repository.
- Changes are committed and pushed to a `civix-upgrade` branch, and a pull request is opened against the repository's default branch.
- Docker logs are collected and uploaded as the `logs.tgz` artifact if the workflow fails.

## CiviCRM PHPUnit Testing Workflow

This reusable GitHub Actions workflow sets up a WordPress + CiviCRM environment and runs an extension's PHPUnit test suite (CiviCRM's headless test framework, PHPUnit 9).

### 📂 File Location

```
.github/workflows/civicrm-phpunit-tests.yml
```

### 🔧 Usage

To use this workflow in another repo, add a workflow such as `.github/workflows/phpunit.yml`:

```yaml
name: PHPUnit Tests

on:
  push
  workflow_dispatch:

jobs:
  phpunit:
    uses: agileware/ci-workflows/.github/workflows/civicrm-phpunit-tests.yml@main
    secrets:
      DOCKERHUB_USER: ${{ secrets.DOCKERHUB_USER }}
      DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
    # Optional
    # with:
    #   EXTENSION_NAME: my_custom_extension
    #   PHPUNIT_ARGS: tests/phpunit/CRM/Foo/BarTest.php
```

The extension must provide a `phpunit.xml.dist` (bootstrapping `tests/phpunit/bootstrap.php`) and tests written against `Civi\Test`'s `HeadlessInterface`, per CiviCRM's [PHPUnit testing docs](https://docs.civicrm.org/dev/en/latest/testing/phpunit/).

### 🔐 Required Secrets

- `DOCKERHUB_USER`
- `DOCKERHUB_TOKEN`

### 🧠 Notes

- If `EXTENSION_NAME` is not provided, the workflow defaults to the name of the calling repository.
- `PHPUNIT_ARGS` is passed straight through to `phpunit9`, e.g. to target a single test file. It defaults to running the full suite from the extension's `phpunit.xml.dist`.
- `phpunit9` is downloaded as an official phar if the base image doesn't already provide it.
- The headless test database is a clone of the main site database (connected to as `root`, since `Civi\Test`'s schema install needs `SUPER`), never the main site DB itself.
- JUnit results are always uploaded as the `phpunit-results` artifact.
- Docker logs are collected and uploaded as the `logs.tgz` artifact if the workflow fails.

## Playwright Frontend Testing Workflow

This reusable GitHub Actions workflow sets up a WordPress + CiviCRM environment, enables the extension under test, and runs its Playwright end-to-end test suite.

### 📂 File Location

```
.github/workflows/playwright-tests.yml
```

### 🔧 Usage

To use this workflow in another repo, add a workflow such as `.github/workflows/playwright.yml`:

```yaml
name: Playwright Tests

on:
  push
  workflow_dispatch:

jobs:
  playwright:
    uses: agileware/ci-workflows/.github/workflows/playwright-tests.yml@main
    secrets:
      DOCKERHUB_USER: ${{ secrets.DOCKERHUB_USER }}
      DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
    # Optional
    # with:
    #   EXTENSION_NAME: my_custom_extension
    #   COMPONENT_TYPE: extension
    #   EXTENSION_KEY: my_custom_extension
    #   PLAYWRIGHT_DIR: tests/playwright
    #   SETUP_SCRIPT: tests/playwright/setup.sh
    #   NODE_VERSION: '20'
```

The component must provide a `package.json` and `playwright.config.ts` (by default under `tests/playwright`), with tests written to run against a live WordPress + CiviCRM site.

#### Testing a WordPress plugin

The same workflow tests a WordPress plugin by setting `COMPONENT_TYPE: plugin`. The repo is then mounted under `wp-content/plugins` and activated with `wp plugin activate` instead of `cv ext:enable`:

```yaml
    with:
      EXTENSION_NAME: wp-civicrm-ux
      COMPONENT_TYPE: plugin
      SETUP_SCRIPT: tests/playwright/fixtures/setup-environment.sh
```

### ⚙️ Inputs

- `EXTENSION_NAME` — name of the component under test, and the folder it's mounted under. Defaults to the calling repository's name.
- `COMPONENT_TYPE` — `extension` (default) or `plugin`. Decides where the repo is mounted and how it is installed: a CiviCRM extension goes under the CiviCRM extensions directory and is enabled with `cv ext:enable`; a WordPress plugin goes under `wp-content/plugins` and is activated with `wp plugin activate`. The default leaves existing callers unchanged.
- `EXTENSION_KEY` — CiviCRM extension key used to enable it via `cv ext:enable`. Defaults to `EXTENSION_NAME`. Ignored when `COMPONENT_TYPE` is `plugin`.
- `PLAYWRIGHT_DIR` — directory (relative to the repo root) containing `package.json` and `playwright.config.ts`. Defaults to `tests/playwright`.
- `SETUP_SCRIPT` — optional path (relative to the repo root) to a repo-specific shell script that runs after WordPress/CiviCRM/the component are up, before the Playwright tests. Use it to create test users/roles and seed data. Runs as `www-data` with its working directory set to the component's folder inside the WordPress container, so it can call `wp` and `cv` directly.
- `NODE_VERSION` — Node.js version to run Playwright with. Defaults to `20`.

### 🔐 Required Secrets

- `DOCKERHUB_USER`
- `DOCKERHUB_TOKEN`

### 🧠 Notes

- The Astra theme is installed and activated, and WordPress permalinks are set to `/%postname%/`, so that CiviCRM's clean URLs (e.g. `/civicrm/dashboard`) resolve as tests expect.
- A CiviCRM WordPress base page (slug `civicrm`) is created and set as the `wpBasePage` setting before rewrite rules are flushed, so CiviCRM's own rewrite rule is included.
- Tests run via `npx playwright test` from `PLAYWRIGHT_DIR`, with `BASE_URL`, `WP_ADMIN_USER`, `WP_ADMIN_PASS` and `CIVI_EXEC_PREFIX` (a ready-to-use `docker exec` prefix for running `wp`/`cv` inside the extension folder) available as environment variables.
- The Playwright HTML report is always uploaded as the `playwright-report` artifact; on failure, screenshots/traces are also uploaded as the `playwright-test-results` artifact.
- Docker logs are collected and uploaded as the `logs.tgz` artifact if the workflow fails.

## WordPress Plugin Cut Release Workflow

This reusable GitHub Actions workflow builds a filtered copy of a WordPress plugin's tree
(dropping tests, CI config, and other dev-only paths) onto a `release` branch, tags that commit,
and publishes the GitHub Release from it.

This exists because a WordPress plugin's update check normally reads whatever commit a version
tag points at (via GitHub's `/releases/latest` API and its `zipball_url`, an archive of that
exact commit). Tagging the plugin's normal development branch directly means client sites
receive its entire tree, tests and CI config included. Tagging a separate, always-filtered
`release` branch instead means client sites only ever receive what they need to run.

CiviCRM extensions have a different release/distribution mechanism (civicrm.org's extension
directory reads git tags directly, with its own versioning conventions), so they need their own
equivalent workflow rather than reusing this one; that hasn't been built yet.

### 📂 File Location

```
.github/workflows/wordpress-plugin-cut-release.yml
```

### 🔧 Usage

The `release` branch must already exist (branched once from the plugin's default branch, with
that repo's excluded paths removed in the first commit) before this workflow is ever run; it
updates the branch, it does not create it.

Add a workflow such as `.github/workflows/cut-release.yml`, triggered manually from the Actions
tab:

```yaml
name: Cut release

on:
  workflow_dispatch:
    inputs:
      VERSION:
        description: 'Version to release (must match the Version header in the plugin file)'
        required: true
        type: string

permissions:
  contents: write

jobs:
  cut-release:
    uses: agileware/ci-workflows/.github/workflows/wordpress-plugin-cut-release.yml@main
    with:
      VERSION: ${{ inputs.VERSION }}
      VERSION_FILE: my-plugin.php
      EXCLUDE_PATHS: "tests .github CONTRIBUTING.md composer.json composer.lock"
      # Optional
      # RELEASE_BRANCH: release
      # PLUGIN_NAME: my-plugin
```

### ⚙️ Inputs

- `VERSION` — version to tag and release, matching the calling repo's existing tag naming (e.g. `2.0.6`).
- `VERSION_FILE` — path (relative to the repo root) to the plugin file whose `Version:` header must equal `VERSION`. The job fails if they don't match, rather than tagging/publishing under the wrong version.
- `EXCLUDE_PATHS` — space-separated repo-relative paths to drop from the release branch. Specific to each plugin's own dev-only paths.
- `RELEASE_BRANCH` — branch that only ever holds filtered, release-ready content. Defaults to `release`. Must already exist.
- `PLUGIN_NAME` — human-readable name, used only in log/commit wording. Defaults to the calling repository's name.

### 🔐 Required Permissions

The calling workflow must declare `permissions: contents: write` itself, in addition to the
reusable workflow doing so: a reusable workflow's effective permissions are the intersection of
both, and this job pushes commits/tags and creates a GitHub Release.

### 🧠 Notes

- Building the release branch resets it to the triggering commit (`git reset --hard`), then
  removes `EXCLUDE_PATHS` as a follow-up commit, rather than copying file contents onto a
  fresh commit on top of release's own history. This is deliberate: `release`'s tip needs
  every commit on the default branch as a real ancestor, or `gh release create
  --generate-notes` (and any other ancestry-based changelog) has nothing to walk and silently
  omits everything merged since release was last cut.
- Because of that reset, updating `release` is pushed with `--force`, it is essentially never
  a fast-forward of release's own previous tip. Nobody should develop directly on `release`,
  its history gets rewritten on every cut.
- If the resulting tree is identical to the release branch's current tip (nothing to release),
  the job still tags and publishes, it just skips creating an empty commit.
- The GitHub Release is created with `--generate-notes`, so it's worth keeping merge commit
  messages/PR titles meaningful on the plugin's default branch.

## CiviCRM Extension Cut Release Workflow

The CiviCRM-extension counterpart to the workflow above. Same idea (a filtered `release` branch
gets tagged instead of the extension's default branch), but versioned from `info.xml` rather
than a plugin file header, since civicrm.org's extension directory reads the tag's tree
directly (no GitHub-specific release-asset or `zipball_url` step to account for).

### 📂 File Location

```
.github/workflows/civicrm-extension-cut-release.yml
```

### 🔧 Usage

The `release` branch must already exist (branched once from the extension's default branch,
with that repo's excluded paths removed in the first commit) before this workflow is ever run;
it updates the branch, it does not create it.

Add a workflow such as `.github/workflows/cut-release.yml`, triggered manually from the Actions
tab:

```yaml
name: Cut release

on:
  workflow_dispatch:
    inputs:
      VERSION:
        description: 'Version to release (must match <version> in info.xml)'
        required: true
        type: string

permissions:
  contents: write

jobs:
  cut-release:
    uses: agileware/ci-workflows/.github/workflows/civicrm-extension-cut-release.yml@main
    with:
      VERSION: ${{ inputs.VERSION }}
      EXCLUDE_PATHS: ".github tests mkdocs.yml"
      # Optional
      # RELEASE_BRANCH: release
      # EXTENSION_NAME: my_extension
```

### ⚙️ Inputs

- `VERSION` — version to tag and release, matching the calling repo's existing tag naming (e.g. `2.0.2`).
- `EXCLUDE_PATHS` — space-separated repo-relative paths to drop from the release branch. Specific to each extension's own dev-only paths.
- `RELEASE_BRANCH` — branch that only ever holds filtered, release-ready content. Defaults to `release`. Must already exist.
- `EXTENSION_NAME` — human-readable name, used only in log/commit wording. Defaults to the calling repository's name.

### 🔐 Required Permissions

Same as the WordPress workflow: the calling workflow must declare `permissions: contents: write`
itself, in addition to the reusable workflow doing so.

### 🧠 Notes

- The version check reads `info.xml`'s `<version>` element, not a plugin file header. Bump
  `<releaseDate>` alongside it, that's on the extension author, this workflow doesn't touch it.
- civicrm.org's directory doesn't require a GitHub Release to exist, it reads the tag directly,
  but one is still published here (`--generate-notes`) to match the existing convention on
  these repos of every tag having a matching Release.
- Otherwise identical to the WordPress workflow: same git-plumbing approach to building the
  filtered tree, same "nothing to release" no-op handling.
