# ci-templates

Shared CI templates for .NET NuGet libraries under [denis-peshkov](https://github.com/denis-peshkov).

The `.NET` pipeline matches the inline CI used by:

- [Cross.Identity](https://github.com/denis-peshkov/Cross.Identity)
- [Cross.CQRS](https://github.com/denis-peshkov/Cross.CQRS)
- [Cross.CQRS.EF](https://github.com/denis-peshkov/Cross.CQRS.EF)

## Repository layout

```text
ci-templates/
├── LICENSE.md
├── README.md
└── .github/
    └── workflows/
        └── templates/
            ├── dotnet-reusable.yml   # workflow_call template (package + description)
            └── dotnet.example.yml    # thin caller example
```

Templates live under **`.github/workflows/templates/`** so they are not picked up as runnable workflows of this repo.

> **Note:** GitHub Actions only treats YAML files **directly** in `.github/workflows/` as workflows / reusable workflows. Files under `templates/` are for **copy / sync** into consumer repos (or promote a copy to `.github/workflows/*.yml` if you want `uses:`).

## Contents

| File | Description |
|------|-------------|
| [`.github/workflows/templates/dotnet-reusable.yml`](.github/workflows/templates/dotnet-reusable.yml) | Shared pipeline (`on: workflow_call`), inputs derived from `package` |
| [`.github/workflows/templates/dotnet.example.yml`](.github/workflows/templates/dotnet.example.yml) | Example thin caller — copy/adapt as consumer `.github/workflows/dotnet.yml` |

## How to consume

### Option A — Copy into the consumer repo (no `uses:`)

1. Copy the reusable file (or sync the `templates/` folder) into the consumer repository.
2. Either:
   - keep a full inline `dotnet.yml` aligned with this template, or
   - place the reusable at **`.github/workflows/dotnet-reusable.yml`** in a shared templates repo (top-level) and call it with `uses:`.

Sync one file from `master`:

```bash
curl -fsSL \
  https://raw.githubusercontent.com/denis-peshkov/ci-templates/master/.github/workflows/templates/dotnet-reusable.yml \
  -o .github/workflows/dotnet-reusable.yml
```

Or sync the whole templates directory via `git subtree` (directory only — not a single file):

```bash
git subtree add --prefix=.github/ci-templates \
  https://github.com/denis-peshkov/ci-templates.git master --squash

git subtree pull --prefix=.github/ci-templates \
  https://github.com/denis-peshkov/ci-templates.git master --squash
```

Then copy/link what you need from `.github/ci-templates/.github/workflows/templates/` into `.github/workflows/`.

### Option B — Reusable workflow `uses:` (requires top-level path)

GitHub `uses:` expects a workflow under `.github/workflows/` (not a nested `templates/` path). To call remotely:

1. Publish/promote `dotnet-reusable.yml` as  
   `.github/workflows/dotnet-reusable.yml` in this repo (top-level), **or**
2. Point `uses:` at whatever top-level path you actually host.

Example caller (after the reusable file is available at top-level):

```yaml
jobs:
  build:
    permissions:
      contents: write
      id-token: write
    uses: denis-peshkov/ci-templates/.github/workflows/dotnet-reusable.yml@master
    with:
      package: 'MyPackage'
      description: 'Short description. Published on NuGet at https://www.nuget.org/packages/MyPackage'
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      TAGTOKEN: ${{ secrets.TAGTOKEN }}
```

See [`.github/workflows/templates/dotnet.example.yml`](.github/workflows/templates/dotnet.example.yml) for triggers + full caller shape.

Prefer pinning `@v1` / a commit SHA once stable instead of `@master`.

## Pipeline

Runs on `ubuntu-22.04`:

1. **Checkout** — `fetch-depth: 0`, `persist-credentials: false` (tag push uses `TAGTOKEN`)
2. **Setup .NET** — SDKs from `dotnet_versions` (default `6.0.x` … `10.0.x`)
3. **Setup NuGet**
4. **GitVersion** — `6.8.2` via `gittools/actions` `@v4.7.0` (`GitVersion.yml` in the consumer repo)
5. **Restore / Build** — MSBuild metadata from inputs + GitVersion
6. **Test** — OpenCover → `TestResults/**/coverage.opencover.xml`
7. **SonarCloud** — quality gate waits only on `pull_request`
8. **Update nuspec** — [`denis-peshkov/update-nuspec-action@v2`](https://github.com/denis-peshkov/update-nuspec-action)
9. **NuGet pack** — symbols from `nuspec_path`
10. **Git tag** — `v{semVer}` only for **stable** SemVer (no `-`) on `master` / `release/*` / `hotfix/*`
11. **NuGet OIDC login + push** — on `master` / `release/*` / `hotfix/*` / `dev` (`NuGet/login@v1`, user `peshkov`)

## Prerequisites (consumer repo)

| Requirement | Notes |
|-------------|--------|
| Layout matching defaults (or overrides) | See [Derived defaults](#derived-defaults-from-package) |
| `GitVersion.yml` at repo root | Used by GitVersion execute |
| SonarCloud project | key/name default to `package` |
| Secret `SONAR_TOKEN` | SonarCloud |
| Secret `TAGTOKEN` | PAT for pushing version tags |
| NuGet.org trusted publishing (OIDC) | User `peshkov` — **no** `NUGET_API_KEY` |

## Recommended triggers (in the consumer)

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

## Derived defaults from `package`

For `package: 'Cross.CQRS.EF'`:

| Value | Default |
|-------|---------|
| `solution` | `Cross.CQRS.EF.slnx` |
| `product` | `Cross.CQRS.EF` |
| `repository_url` | `https://github.com/{github.repository_owner}/Cross.CQRS.EF.git` |
| `sonar_organization` | `denis-peshkov` |
| `sonar_project_key` | `Cross.CQRS.EF` |
| `sonar_project_name` | `Cross.CQRS.EF` |
| `sonar_sources` | `Cross.CQRS.EF/` |
| `sonar_tests` | `Cross.CQRS.EF.Tests/` |
| `nuspec_path` | `Cross.CQRS.EF/config.nuspec` |
| `package_name` | `Cross.CQRS.EF` |

Override any of these with the same-named input when the layout differs.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `package` | yes | — | Package / project id; drives derived paths |
| `description` | yes | — | MSBuild `Description` |
| `solution` | no | `{package}.slnx` | Solution file |
| `product` | no | `{package}` | MSBuild `Product` |
| `repository_url` | no | `https://github.com/{owner}/{package}.git` | MSBuild `RepositoryUrl` |
| `sonar_organization` | no | `denis-peshkov` | SonarCloud organization |
| `sonar_project_key` | no | `{package}` | SonarCloud project key |
| `sonar_project_name` | no | `{package}` | SonarCloud project display name |
| `sonar_sources` | no | `{package}/` | Sources path |
| `sonar_tests` | no | `{package}.Tests/` | Tests path |
| `sonar_cpd_exclusions` | no | `''` | Optional CPD exclusions; omitted when empty |
| `nuspec_path` | no | `{package}/config.nuspec` | Path to `config.nuspec` |
| `package_name` | no | `{package}` | Package id for `nuget push` glob |
| `build_config` | no | `Release` | MSBuild configuration |
| `dotnet_versions` | no | `6.0.x`…`10.0.x` | Multiline list for `actions/setup-dotnet` |

### Optional overrides

**Cross.CQRS** — include `3.1.x`:

```yaml
dotnet_versions: |
  3.1.x
  6.0.x
  7.0.x
  8.0.x
  9.0.x
  10.0.x
```

**Cross.Identity** — CPD exclusions:

```yaml
sonar_cpd_exclusions: '**/ProcessEngine/Definitions/Templates/**'
```

**Non-standard layout:**

```yaml
solution: 'Cross.Json.sln'
nuspec_path: '_nuget/config.nuspec'
sonar_tests: 'FunctionalTests/'
```

## Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `SONAR_TOKEN` | yes | SonarCloud token |
| `TAGTOKEN` | yes | GitHub PAT for pushing version tags |

NuGet publishing uses OIDC (`NuGet/login@v1`). Do **not** pass `NUGET_API_KEY`.

## Publish / tag matrix

| Ref | Build / test / Sonar | Quality gate wait | Git tag | NuGet push |
|-----|----------------------|-------------------|---------|------------|
| Pull request | yes | yes | no | no |
| `feature/*` / `fix/*` / `chore/*` | yes | no | no | no |
| `dev` | yes | no | no | yes |
| `master` / `release/*` / `hotfix/*` | yes | no | yes if SemVer has no `-` | yes |

## Out of scope

- Multi-package pack (e.g. Cross.PepperVault)
- VSIX/Rider (TypeScriptDefinitionGenerator)
- App deploy pipelines, Rust CLIs

## Related

- [update-nuspec-action](https://github.com/denis-peshkov/update-nuspec-action)
- [Building and testing .NET](https://docs.github.com/en/actions/automating-builds-and-tests/building-and-testing-net)
- [Reusing workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)

## License

[MIT](LICENSE.md)
