# shared-workflows

Reusable GitHub Actions workflows for the `project-apricot` org.

```yaml
uses: project-apricot/shared-workflows/.github/workflows/<file>@v1
```

Pin `@v1` — a moving tag tracking the latest backwards-compatible release. Breaking input
changes ship as `v2`, so repos migrate on their own schedule.

## Workflows

| File | Purpose |
| --- | --- |
| `dotnet-lib-ci.yml` | Restore, build, test and pack. Publishes nothing. |
| `release-prepare.yml` | Infer the next version from conventional commits, create the GitHub Release. |
| `dotnet-lib-preview.yml` | Publish a prerelease to GitHub Packages, on demand. |
| `pr-conventional-title.yml` | Enforce conventional-commit PR titles. |
| `version-semantic.yml` | Versioning primitive, ecosystem-agnostic. Used by the above; rarely called directly. |

## Actions

| Path | Purpose |
| --- | --- |
| `actions/dotnet-publish` | Build, test, pack and push a tagged library to nuget.org. |

**Publishing to nuget.org must be a composite action, not a reusable workflow.** nuget.org
validates the OIDC claim `job_workflow_ref` and requires it to start with
`<owner>/<repo>/.github/workflows/`. A reusable workflow in this repo sets that claim to
*its own* path, so the token exchange fails with HTTP 401 and no policy can allow it.
Composite action steps run inside the caller's job, so the claim stays the calling
repository's workflow. The calling job must grant `id-token: write` and check out the tag.

Inputs, secrets and required caller permissions are documented in each file. GitHub only
resolves reusable workflows directly under `.github/workflows/`, so the taxonomy lives in
the filenames: `<ecosystem>-<artifact>-<action>.yml`.

## Release model

The trigger is the signal — nothing declares "this is a real release":

| Trigger in the calling repo | Result |
| --- | --- |
| `pull_request`, `push` | validation only |
| dispatch `release-prepare` | tag + GitHub Release |
| `release: [published]` | publish to nuget.org (a local job using `actions/dotnet-publish`) |
| dispatch `preview` | prerelease to GitHub Packages |

A release is therefore two auditable steps: dispatch `release-prepare`, and the Release it
creates triggers the publish. The tag is the version; nobody types one by hand.

## Releasing this repo

Dispatch **Self release** (`self-release.yml`). It cuts `vX.Y.Z` with generated notes and
moves the major tag that callers pin — both halves matter, since a new version nobody is
pinned to changes nothing.

Equivalent by hand, if `release-prepare.yml` is what you just broke:

```bash
git tag v1.2.0 && git push origin v1.2.0
git tag -f v1 v1.2.0 && git push -f origin v1
```

Test a change first by pointing a caller's `uses:` at a branch ref, with `dry_run: true`
where supported.
