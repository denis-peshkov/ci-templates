# ci-templates

Shared GitHub Actions reusable workflows for .NET NuGet libraries.

Etalon is based on:

- [Cross.Identity](https://github.com/denis-peshkov/Cross.Identity)
- [Cross.CQRS](https://github.com/denis-peshkov/Cross.CQRS)
- [Cross.CQRS.EF](https://github.com/denis-peshkov/Cross.CQRS.EF)

## Contents

| File | Description |
|------|-------------|
| [`.github/workflows/dotnet-reusable.yml`](.github/workflows/dotnet-reusable.yml) | Reusable workflow |
| [`.github/workflows/dotnet.example.yml`](.github/workflows/dotnet.example.yml) | Caller example (triggers + `uses:`) |

## Pipeline

1. Checkout (`fetch-depth: 0`, `persist-credentials: false`)
2. Setup .NET / NuGet
3. GitVersion `6.8.2` via `gittools/actions` `v4.7.0`
4. Restore, build, test (OpenCover)
5. SonarCloud (quality gate waits on pull requests)
6. `update-nuspec-action@v2`
7. `nuget pack` (symbols)
8. Git tag `v{semVer}` — only **stable** SemVer on `master` / `release/*` / `hotfix/*`
9. NuGet OIDC login + push — on `master` / `release/*` / `hotfix/*` / `dev`

## Quick start

Copy [`.github/workflows/dotnet.example.yml`](.github/workflows/dotnet.example.yml) to your repo as `.github/workflows/dotnet.yml` and adjust `with:`.

```yaml
jobs:
  build:
    permissions:
      contents: write
      id-token: write
    uses: denis-peshkov/ci-templates/.github/workflows/dotnet-reusable.yml@master
    with:
      solution: 'MyPackage.slnx'
      product: 'MyPackage'
      description: '...'
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

For `Cross.CQRS` pass `dotnet_versions` including `3.1.x`.

For `Cross.Identity` CPD exclusions:

```yaml
sonar_cpd_exclusions: '**/ProcessEngine/Definitions/Templates/**'
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `solution` | yes | — | Solution file |
| `product` | yes | — | MSBuild `Product` |
| `description` | yes | — | MSBuild `Description` |
| `repository_url` | yes | — | MSBuild `RepositoryUrl` |
| `sonar_organization` | no | `denis-peshkov` | SonarCloud org |
| `sonar_project_key` | yes | — | SonarCloud project key |
| `sonar_project_name` | yes | — | SonarCloud project name |
| `sonar_sources` | yes | — | Sources path |
| `sonar_tests` | yes | — | Tests path |
| `sonar_cpd_exclusions` | no | `''` | Optional CPD exclusions |
| `nuspec_path` | yes | — | Path to `config.nuspec` |
| `package_name` | yes | — | Package id for push glob |
| `build_config` | no | `Release` | Configuration |
| `dotnet_versions` | no | `6.0.x`…`10.0.x` | SDK list |

## Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `SONAR_TOKEN` | yes | SonarCloud |
| `TAGTOKEN` | yes | PAT for git tag push |

NuGet publish uses OIDC (`NuGet/login@v1`, user `peshkov`) — no `NUGET_API_KEY` secret.

## Caller triggers (recommended)

```yaml
on:
  push:
    branches: [master, release/*, hotfix/*, dev, feature/*, fix/*, chore/*]
  pull_request:
    types: [opened, synchronize, reopened]
    branches: [master, release/*, hotfix/*, dev]
  workflow_dispatch:
```

## Publish / tag matrix

| Branch | Tag (stable only) | NuGet push |
|--------|-------------------|------------|
| `master` / `release/*` / `hotfix/*` | yes if SemVer has no `-` | yes |
| `dev` | no | yes |
| `feature/*` / `fix/*` / `chore/*` / PR | no | no |

## License

[MIT](LICENSE)
