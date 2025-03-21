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
