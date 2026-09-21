# CI Workflows

Reusable GitHub Actions workflows for Agileware's CiviCRM extension repos. Each
workflow spins up a WordPress + CiviCRM environment in Docker and runs a
specific task against it.

- [civix Upgrade Workflow](#civix-upgrade-workflow) — runs `civix upgrade` and opens a PR with the result
- [CiviCRM PHPUnit Testing Workflow](#civicrm-phpunit-testing-workflow) — runs an extension's headless PHPUnit suite
- [Playwright Frontend Testing Workflow](#playwright-frontend-testing-workflow) — runs an extension's Playwright test suite

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
    #   EXTENSION_KEY: my_custom_extension
    #   PLAYWRIGHT_DIR: tests/playwright
    #   SETUP_SCRIPT: tests/playwright/setup.sh
    #   NODE_VERSION: '20'
```

The extension must provide a `package.json` and `playwright.config.ts` (by default under `tests/playwright`), with tests written to run against a live WordPress + CiviCRM site.

### ⚙️ Inputs

- `EXTENSION_NAME` — name of the CiviCRM extension, and the folder it's mounted under. Defaults to the calling repository's name.
- `EXTENSION_KEY` — CiviCRM extension key used to enable it via `cv ext:enable`. Defaults to `EXTENSION_NAME`.
- `PLAYWRIGHT_DIR` — directory (relative to the repo root) containing `package.json` and `playwright.config.ts`. Defaults to `tests/playwright`.
- `SETUP_SCRIPT` — optional path (relative to the repo root) to a repo-specific shell script that runs after WordPress/CiviCRM/the extension are up, before the Playwright tests. Use it to create test users/roles and seed data. Runs as `www-data` with its working directory set to the extension folder inside the WordPress container, so it can call `wp` and `cv` directly.
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
