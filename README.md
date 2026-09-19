# ci-templates

Shared [GitHub Actions](https://docs.github.com/en/actions) reusable workflows for .NET NuGet libraries under [denis-peshkov](https://github.com/denis-peshkov).

The `.NET` etalon matches the inline CI used by:

- [Cross.Identity](https://github.com/denis-peshkov/Cross.Identity)
- [Cross.CQRS](https://github.com/denis-peshkov/Cross.CQRS)
- [Cross.CQRS.EF](https://github.com/denis-peshkov/Cross.CQRS.EF)

Caller repositories pass mainly `package` + `description`. Paths and Sonar/NuGet ids are derived unless overridden.

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
| Layout matching defaults (or overrides) | See [Derived defaults](#derived-defaults-from-package) |
| `GitVersion.yml` at repo root | Used by GitVersion execute |
| SonarCloud project | key/name default to `package` |
| Secret `SONAR_TOKEN` | SonarCloud |
| Secret `TAGTOKEN` | PAT that can push tags (ruleset bypass if needed) |
| NuGet.org trusted publishing (OIDC) | For user `peshkov` — **no** `NUGET_API_KEY` secret |
| Access to this repo | Public, or Actions access granted for private callers |

## Quick start

1. Copy [`.github/workflows/dotnet.example.yml`](.github/workflows/dotnet.example.yml) → `.github/workflows/dotnet.yml`.
2. Set `package` and `description` (add overrides only if the layout differs).
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
      package: 'MyPackage'
      description: 'Short description. Published on NuGet at https://www.nuget.org/packages/MyPackage'
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

Any of these can be overridden via the same-named input.

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

### Optional overrides (examples)

**Cross.CQRS** — include netcoreapp3.1 SDK:

```yaml
with:
  package: 'Cross.CQRS'
  description: '...'
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
with:
  package: 'Cross.Identity'
  description: '...'
  sonar_cpd_exclusions: '**/ProcessEngine/Definitions/Templates/**'
```

**Non-standard layout** (old repos):

```yaml
with:
  package: 'Cross.Json'
  description: '...'
  solution: 'Cross.Json.sln'
  nuspec_path: '_nuget/config.nuspec'
  sonar_tests: ''   # only if you must override; prefer a real tests path when present
```

`sonar_cpd_exclusions` maps to `-Dsonar.cpd.exclusions=...` (Sonar Copy/Paste Detection).

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

1. Keep `on:` in the consumer repo (align with the recommended triggers if needed).
2. Replace the job body with `uses:` + `with:` (`package`, `description`, rare overrides) + `secrets:` + `permissions`.
3. Verify on a PR first, then on `dev` / `master` as needed.

### Suggested migration order

1. Push/tag this `ci-templates` repo.
2. Pilot: [Cross.CQRS.EF](https://github.com/denis-peshkov/Cross.CQRS.EF) (`package` + `description` only).
3. [Cross.Identity](https://github.com/denis-peshkov/Cross.Identity) (`sonar_cpd_exclusions`) and [Cross.CQRS](https://github.com/denis-peshkov/Cross.CQRS) (`dotnet_versions` with `3.1.x`).
4. Other single-package NuGet repos that match `{package}.slnx` / `{package}/` / `{package}.Tests/` / `{package}/config.nuspec`.
5. Older layouts via overrides (`*.sln`, `_nuget/config.nuspec`, `*.UnitTests/`, `src/`/`test/`).
6. **Out of scope** for this workflow: multi-package pack (e.g. Cross.PepperVault), VSIX/Rider (TypeScriptDefinitionGenerator), deploy apps (peshkov.biz), Rust CLIs.

## Related

- [update-nuspec-action](https://github.com/denis-peshkov/update-nuspec-action)
- [Building and testing .NET](https://docs.github.com/en/actions/automating-builds-and-tests/building-and-testing-net)
- [Reusing workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)

## License

[MIT](LICENSE)
