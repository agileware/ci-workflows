# Civix Upgrade Workflow

This reusable GitHub Actions workflow sets up a WordPress + CiviCRM environment and runs `civix upgrade` on a CiviCRM extension.

## 📂 File Location

```
.civix-upgrade.yml
```

## 🔧 Usage

To use this workflow in another repo, add the following to `.github/workflows/upgrade.yml`:

```yaml
name: Upgrade Extension

on:
  push

jobs:
  upgrade:
    uses: agileware/ci-workflows/civix-upgrade-template-final.yml@main
    secrets:
      DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
      DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
    # Optional
    # inputs:
    #   EXTENSION_NAME: my_custom_extension
```

## 🔐 Required Secrets

- `DOCKERHUB_USER`
- `DOCKERHUB_TOKEN`

## 🧠 Notes

- If `EXTENSION_NAME` is not provided, the workflow defaults to the name of the calling repository.

# CiviCRM PHPUnit Testing Workflow

This reusable GitHub Actions workflow sets up a WordPress + CiviCRM environment and runs an extension's PHPUnit test suite (CiviCRM's headless test framework, PHPUnit 9).

## 📂 File Location

```
.github/workflows/civicrm-phpunit-tests.yml
```

## 🔧 Usage

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

## 🔐 Required Secrets

- `DOCKERHUB_USER`
- `DOCKERHUB_TOKEN`

## 🧠 Notes

- If `EXTENSION_NAME` is not provided, the workflow defaults to the name of the calling repository.
- `PHPUNIT_ARGS` is passed straight through to `phpunit9`, e.g. to target a single test file. It defaults to running the full suite from the extension's `phpunit.xml.dist`.
- `phpunit9` is downloaded as an official phar if the base image doesn't already provide it.
- JUnit results are always uploaded as the `phpunit-results` artifact.
