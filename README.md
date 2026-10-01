# CI Workflows

Reusable GitHub Actions workflows for Agileware's CiviCRM extension and WordPress plugin
repos. Most workflows spin up a WordPress + CiviCRM environment in Docker and run a specific
task against it; the release workflows operate on git/GitHub directly instead.

- [civix Upgrade Workflow](#civix-upgrade-workflow) — runs `civix upgrade` and opens a PR with the result
- [CiviCRM PHPUnit Testing Workflow](#civicrm-phpunit-testing-workflow) — runs an extension's headless PHPUnit suite
- [Playwright Frontend Testing Workflow](#playwright-frontend-testing-workflow) — runs an extension's Playwright test suite
- [WordPress Plugin Cut Release Workflow](#wordpress-plugin-cut-release-workflow) — builds a filtered release branch, tags it, and publishes the GitHub Release
- [CiviCRM Extension Cut Release Workflow](#civicrm-extension-cut-release-workflow) — same, versioned from info.xml instead of a plugin header
- [CiviCRM Extension Init Release Workflow](#civicrm-extension-init-release-workflow) — one-time setup that adds the Cut Release workflow and `release` branch to a CiviCRM extension repo

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
- `PRE_ACTIVATE_SCRIPT` — optional path (relative to the repo root) to a repo-specific shell script that runs after WordPress and CiviCRM are installed but *before* the component is activated or enabled. Use it to install plugins the component depends on: since WordPress 6.5, `wp plugin activate` refuses a plugin whose `Requires plugins:` are not active. Runs as `www-data` in the component's folder, like `SETUP_SCRIPT`, and receives the `GRAVITYFORMS_LICENSE_KEY` secret (if passed) as an environment variable.

### 🔐 Required Secrets

- `DOCKERHUB_USER`
- `DOCKERHUB_TOKEN`

Optional:

- `GRAVITYFORMS_LICENSE_KEY` — for callers whose `PRE_ACTIVATE_SCRIPT` installs Gravity Forms or its add-ons, which are commercially licensed. Passed to the script as an environment variable and never placed on a command line. Pull requests from forks do not receive secrets.

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
- `phpunit.xml.dist` is always stripped too, regardless of `EXCLUDE_PATHS`: it's PHPUnit's own
  test-suite config, useless without `tests/` (already excluded by convention), and has no
  runtime purpose on a client site.
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

Before building the release branch, this workflow also standardises the extension's
documentation structure on the calling branch itself (`mkdocs.yml`, a `docs/` directory, an
absolute-linked `docs/README.md`, `docs/logo/agileware-logo.png`, and `info.xml`'s
Documentation URL), pushing that as a normal commit before the filtered `release` branch is
built from it. See [Documentation Structure](#documentation-structure) below.

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
      EXCLUDE_PATHS: ".github tests"
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
- `mkdocs.yml` must **not** be in `EXCLUDE_PATHS`: it ships in the release because the mkdocs
  server builds/serves the documentation site directly from it, same as `docs/` itself.

### Documentation Structure

Every cut-release run checks the calling branch (not the `release` branch) for the standard
docs layout used across these extensions (see `au.com.agileware.eventmanagelocations` and
`au.com.agileware.ewayrecurring`), and brings it up to date before the release branch is built
from it:

1. Creates `mkdocs.yml` if missing, filling in `site_name`/`repo_url`/`site_description` from
   `info.xml`'s `<name>` element and the calling repository, from the template in
   [ci-workflows' own `docs-template/`](docs-template/mkdocs.yml).
2. Creates `docs/` if missing.
3. Moves `README.md` into `docs/README.md` if the latter doesn't exist yet (a move, not a copy,
   there's no reason to keep two diverging copies once `docs/README.md` is the published one).
4. Rewrites every relative link/image in `docs/README.md` to an absolute
   `https://github.com/<repo>/blob/<default-branch>/...` (or `.../raw/...` for images) URL.
   mkdocs only serves files under `docs/`, so a relative link that resolves correctly when
   GitHub renders the file in place breaks once the same content is built into a standalone
   mkdocs site. A README just moved from the repo root (step 3, this run) has links written
   relative to the root; one that already lived in `docs/` (a prior run, or a repo that started
   there, like `ewayrecurring`) has links written relative to `docs/` itself, this step resolves
   against whichever base actually applies.
5. Creates `docs/logo/agileware-logo.png` if missing, copied from
   [ci-workflows' `docs-template/logo/`](docs-template/logo/agileware-logo.png), not from
   anything already in the calling repo, so every extension ends up with the same asset. If
   this step actually creates it (i.e. it was missing), and an old root-level
   `logo/agileware-logo.png` exists (from before this extension had a `docs/` structure), that
   old file is removed too, it's unreferenced the moment `docs/README.md`'s own logo link
   points at `docs/logo/` instead. The directory goes too, but only once it's empty, in case
   some repo's root `logo/` ever holds something else alongside it.
6. Updates (or adds) `info.xml`'s `Documentation` URL to point at `docs/README.md`, since that's
   civicrm.org's extension directory's own source for that link, and it needs to follow the move
   in step 3.

Each check is independent and idempotent, a repo that already has some or all of this (like
`ewayrecurring`) only gets the parts it's actually missing brought up to date, and a repo that's
fully up to date produces no commit at all. Verified, before this was ever used for real, by
extracting each step's script and running it against: a throwaway repo seeded with a real
extension's pre-migration `README.md`/`info.xml` (reproduced `au.com.agileware.ewayrecurring`'s
real historical "Use absolute GitHub URLs" commit byte for byte), and a throwaway repo seeded
with `au.com.agileware.ewayrecurring`'s current, already-migrated `docs/`, which came back
unchanged.

## CiviCRM Extension Init Release Workflow

A one-time setup workflow for a CiviCRM extension repo that doesn't have the Cut Release
workflow yet. Preparing a repo for `civicrm-extension-cut-release.yml` by hand means writing
its caller workflow, writing a `CONTRIBUTING.md` process note, branching/stripping/pushing the
initial `release` branch, and adding whichever of this org's other reusable CI workflows
(`civix-upgrade.yml`, `civicrm-phpunit-tests.yml`, `playwright-tests.yml`) the repo doesn't
already have a caller for, this does all of that in one dispatch.

### 📂 File Location

```
.github/workflows/civicrm-extension-init-release.yml
```

### 🔧 Usage

Add a workflow such as `.github/workflows/init-release.yml`, triggered manually from the
Actions tab. Unlike the Cut Release caller, this file never needs editing per repo, the exclude
list is supplied at dispatch time instead of baked in:

```yaml
name: Init release

on:
  workflow_dispatch:
    inputs:
      EXCLUDE_PATHS:
        description: 'Space-separated repo-relative paths to drop from the release branch (e.g. ".github tests .idea")'
        required: true
        type: string

permissions:
  contents: write
  workflows: write

jobs:
  init-release:
    uses: agileware/ci-workflows/.github/workflows/civicrm-extension-init-release.yml@main
    with:
      EXCLUDE_PATHS: ${{ inputs.EXCLUDE_PATHS }}
```

Run it once from the Actions tab with the exclude paths for that repo (inspect the repo first,
same judgement call as before: dev-only CI config, test suites, IDE folders like `.idea`, but
not `mkdocs.yml` or an extension's own shipped `docs/`, both of which ship in the release). It
then:

1. Writes `.github/workflows/cut-release.yml` (pre-filled with the `EXCLUDE_PATHS` you passed)
   and a `CONTRIBUTING.md` process note.
2. Writes `.github/workflows/civix-upgrade.yml` if the repo doesn't already have one, so every
   extension gets `civix upgrade` PRs opened automatically.
3. Writes `.github/workflows/phpunit-tests.yml` if the repo has a `phpunit.xml.dist` at its
   root and doesn't already have this workflow, since there's nothing to test otherwise.
4. Writes `.github/workflows/frontend-tests.yml` if the repo has
   `tests/playwright/playwright.config.ts` and doesn't already have this workflow, same
   reasoning. Also wires in `SETUP_SCRIPT: tests/playwright/fixtures/setup-environment.sh`
   when that file exists too, matching the convention already used elsewhere, `SETUP_SCRIPT`
   is otherwise left out rather than guessed at.
5. Commits whatever combination of the above was actually added, plus `CONTRIBUTING.md`, in one
   commit, and pushes straight to the calling branch.
6. Branches `release` from that commit, strips `EXCLUDE_PATHS` from it, and pushes it as a new
   branch on origin.

Steps 2-4 never touch a file that's already there, re-running this workflow (it refuses to,
per the double-init guard below, but the individual file-writing steps are harmless no-ops on
their own too) never overwrites a hand-tuned existing workflow.

After it finishes, the repo is ready for `civicrm-extension-cut-release.yml` exactly as if it
had been set up by hand, dispatch "Cut release" the normal way to test it.

### ⚙️ Inputs

- `EXCLUDE_PATHS` — space-separated repo-relative paths to drop from the release branch. Baked
  verbatim into the generated `cut-release.yml` and into the `CONTRIBUTING.md` note.
- `RELEASE_BRANCH` — name for the branch this workflow creates. Defaults to `release`. Must not
  already exist, this workflow creates it, it doesn't update one.
- `EXTENSION_NAME` — human-readable name, used only in log wording. Defaults to the calling
  repository's name.

### 🔐 Required Permissions

The calling workflow must declare both `permissions: contents: write` and
`permissions: workflows: write` itself, in addition to the reusable workflow doing so: writing a
new file under `.github/workflows/` needs the `workflows` permission as well as `contents`, a
reusable workflow's effective permissions are the intersection of both.

### 🧠 Notes

- Refuses to run if `info.xml` is missing (not a CiviCRM extension), if
  `.github/workflows/cut-release.yml` already exists, or if `RELEASE_BRANCH` already exists on
  origin, so it's safe to leave this workflow file in a repo permanently without risking a
  double-init.
- The generated `cut-release.yml` contains a literal `${{ inputs.VERSION }}` expression, written
  via GitHub's documented `${{ '${{' }}` escaping trick so that this workflow's own templating
  pass doesn't try to evaluate it (this workflow has no `VERSION` input to evaluate it against).
  Verified by extracting and running each `run:` step's script directly against a throwaway
  local repo before this workflow was first used for real.
- Same git-plumbing approach as the Cut Release workflows otherwise: a straightforward push for
  the first commit on the calling branch, then a fresh `release` branch built directly from it
  (no reset/force-push needed yet, there's no prior `release` history to reconcile).
