# ci-templates

Shared [GitHub Actions](https://docs.github.com/en/actions) reusable workflows for .NET NuGet libraries under [denis-peshkov](https://github.com/denis-peshkov).

The `.NET` etalon matches the inline CI used by:

- [Cross.Identity](https://github.com/denis-peshkov/Cross.Identity)
- [Cross.CQRS](https://github.com/denis-peshkov/Cross.CQRS)
- [Cross.CQRS.EF](https://github.com/denis-peshkov/Cross.CQRS.EF)

Caller repositories keep only triggers + project-specific `with:` / `secrets:`. Shared steps live here.

## Contents

| File | Description |
|------|-------------|
| [`.github/workflows/dotnet-reusable.yml`](.github/workflows/dotnet-reusable.yml) | Reusable workflow (`workflow_call`) |
| [`.github/workflows/dotnet.example.yml`](.github/workflows/dotnet.example.yml) | Full caller example — copy as `.github/workflows/dotnet.yml` |

## Pipeline

Runs on `ubuntu-22.04`:

1. **Checkout** — `fetch-depth: 0`, `persist-credentials: false` (so tag push uses `TAGTOKEN`)
2. **Setup .NET** — SDKs from `dotnet_versions` (default `6.0.x` … `10.0.x`)
3. **Setup NuGet**
4. **GitVersion** — `6.8.2` via `gittools/actions` `@v4.7.0` (reads `GitVersion.yml` in the caller repo)
5. **Restore / Build** — MSBuild metadata from inputs + GitVersion env vars
6. **Test** — OpenCover → `TestResults/**/coverage.opencover.xml`
7. **SonarCloud** — quality gate waits only on `pull_request`
8. **Update nuspec** — [`denis-peshkov/update-nuspec-action@v2`](https://github.com/denis-peshkov/update-nuspec-action)
9. **NuGet pack** — symbols package from `nuspec_path`
10. **Git tag** — `v{semVer}` only for **stable** SemVer (no `-`) on `master` / `release/*` / `hotfix/*`
11. **NuGet OIDC login + push** — on `master` / `release/*` / `hotfix/*` / `dev` (`NuGet/login@v1`, user `peshkov`)

## Prerequisites (caller repo)

| Requirement | Notes |
|-------------|--------|
| `GitVersion.yml` at repo root | Used by GitVersion execute |
| `config.nuspec` | Path passed as `nuspec_path` |
| SonarCloud project | `sonar_project_key` / `sonar_project_name` must match |
| Secret `SONAR_TOKEN` | SonarCloud |
| Secret `TAGTOKEN` | PAT that can push tags (ruleset bypass if needed) |
| NuGet.org trusted publishing (OIDC) | For user `peshkov` — **no** `NUGET_API_KEY` secret |
| Access to this repo | Public, or Actions access granted for private callers |

## Quick start

1. Copy [`.github/workflows/dotnet.example.yml`](.github/workflows/dotnet.example.yml) → `.github/workflows/dotnet.yml` in the consumer repo.
2. Replace project-specific `with:` values.
3. Prefer pinning a tag/SHA instead of `@master` once stable:

```yaml
uses: denis-peshkov/ci-templates/.github/workflows/dotnet-reusable.yml@v1
```

### Minimal caller job

```yaml
jobs:
  build:
    permissions:
      contents: write   # git tag push
      id-token: write   # NuGet OIDC
    uses: denis-peshkov/ci-templates/.github/workflows/dotnet-reusable.yml@master
    with:
      solution: 'MyPackage.slnx'
      product: 'MyPackage'
      description: 'Short description. Published on NuGet at https://www.nuget.org/packages/MyPackage'
      repository_url: 'https://github.com/denis-peshkov/MyPackage.git'
      sonar_project_key: 'MyPackage'
      sonar_project_name: 'MyPackage'
      sonar_sources: 'MyPackage/'
      sonar_tests: 'MyPackage.Tests/'
      nuspec_path: 'MyPackage/config.nuspec'
      package_name: 'MyPackage'
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      TAGTOKEN: ${{ secrets.TAGTOKEN }}
```

`permissions` and `secrets` must be set on the **caller** job. Reusable workflows do not inherit repository secrets automatically.

### Recommended triggers (in the caller)

Reusable workflows do **not** define `on:` — keep this in each repo:

```yaml
on:
  push:
    branches:
      - master
      - release/*
      - hotfix/*
      - dev
      - feature/*
      - fix/*
      - chore/*
  pull_request:
    types: [opened, synchronize, reopened]
    branches:
      - master
      - release/*
      - hotfix/*
      - dev
  workflow_dispatch:
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `solution` | yes | — | Solution file (e.g. `Cross.Identity.slnx`) |
| `product` | yes | — | MSBuild `Product` |
| `description` | yes | — | MSBuild `Description` |
| `repository_url` | yes | — | MSBuild `RepositoryUrl` (`.git` URL) |
| `sonar_organization` | no | `denis-peshkov` | SonarCloud organization |
| `sonar_project_key` | yes | — | SonarCloud project key |
| `sonar_project_name` | yes | — | SonarCloud project display name |
| `sonar_sources` | yes | — | Sources path (trailing `/` recommended) |
| `sonar_tests` | yes | — | Tests path (trailing `/` recommended) |
| `sonar_cpd_exclusions` | no | `''` | Optional CPD exclusions; omitted from Sonar args when empty |
| `nuspec_path` | yes | — | Path to `config.nuspec` |
| `package_name` | yes | — | Package id used in `nuget push **/Name.{semVer}.symbols.nupkg` |
| `build_config` | no | `Release` | MSBuild configuration |
| `dotnet_versions` | no | `6.0.x`…`10.0.x` | Multiline list for `actions/setup-dotnet` |

### Optional inputs (examples)

**Cross.CQRS** — include netcoreapp3.1 SDK:

```yaml
dotnet_versions: |
  3.1.x
  6.0.x
  7.0.x
  8.0.x
  9.0.x
  10.0.x
```

**Cross.Identity** — skip copy/paste noise on process templates:

```yaml
sonar_cpd_exclusions: '**/ProcessEngine/Definitions/Templates/**'
```

`sonar_cpd_exclusions` maps to `-Dsonar.cpd.exclusions=...` (Sonar Copy/Paste Detection). Leave unset for most packages.

## Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `SONAR_TOKEN` | yes | SonarCloud token |
| `TAGTOKEN` | yes | GitHub PAT for pushing version tags |

NuGet publishing uses OIDC temporary credentials (`NuGet/login@v1`). Do **not** pass `NUGET_API_KEY`.

## Publish / tag matrix

| Ref | Build / test / Sonar | Quality gate wait | Git tag | NuGet push |
|-----|----------------------|-------------------|---------|------------|
| Pull request | yes | yes | no | no |
| `feature/*` / `fix/*` / `chore/*` | yes | no | no | no |
| `dev` | yes | no | no | yes |
| `master` / `release/*` / `hotfix/*` | yes | no | yes if SemVer has no `-` | yes |

## Migrating an existing `dotnet.yml`

1. Keep `on:` in the consumer repo (align with the recommended triggers above if needed).
2. Replace the job body with `uses:` + `with:` + `secrets:` + `permissions`.
3. Map hardcoded values:

| Former inline value | Caller `with:` |
|---------------------|----------------|
| `SOLUTION` / env | `solution` |
| `-p:Product=` | `product` |
| `-p:Description=` | `description` |
| `-p:RepositoryUrl=` | `repository_url` |
| `-Dsonar.projectKey=` | `sonar_project_key` |
| `-Dsonar.projectName=` | `sonar_project_name` |
| `-Dsonar.sources=` / `tests=` | `sonar_sources` / `sonar_tests` |
| `-Dsonar.cpd.exclusions=` | `sonar_cpd_exclusions` (optional) |
| `nuget pack path/config.nuspec` | `nuspec_path` |
| push glob package id | `package_name` |
| extra SDKs (e.g. `3.1.x`) | `dotnet_versions` |

4. Verify on a PR first, then on `dev` / `master` as needed.

### Suggested migration order

1. Push/tag this `ci-templates` repo.
2. Pilot: [Cross.CQRS.EF](https://github.com/denis-peshkov/Cross.CQRS.EF) (no extra inputs).
3. [Cross.Identity](https://github.com/denis-peshkov/Cross.Identity) (`sonar_cpd_exclusions`) and [Cross.CQRS](https://github.com/denis-peshkov/Cross.CQRS) (`dotnet_versions` with `3.1.x`).
4. Other single-package NuGet repos.
5. **Out of scope** for this workflow: multi-package pack (e.g. Cross.PepperVault), VSIX/Rider (TypeScriptDefinitionGenerator), deploy apps (peshkov.biz), Rust CLIs.

## Related

- [update-nuspec-action](https://github.com/denis-peshkov/update-nuspec-action)
- [Building and testing .NET](https://docs.github.com/en/actions/automating-builds-and-tests/building-and-testing-net)
- [Reusing workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)

## License

[MIT](LICENSE)
