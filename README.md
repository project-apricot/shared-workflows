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
| `docs-release.yml` | Mirror a library's `docs/` into the documentation site as a pull request. |

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
| dispatch `docs` | mirror `docs/` into the documentation site |

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

## Documentation

`docs-release.yml` mirrors a library's `docs/` folder into
[`project-apricot/docs`](https://github.com/project-apricot/docs), which builds
projectapricot.dev. The library owns its docs; the site owns presentation and never edits the
files it vendors, so the mirror is byte-for-byte and `--delete`d — a page removed upstream
disappears from the site.

Two callers per library:

```yaml
# docs.yml — standalone, when only the docs changed
on: { workflow_dispatch: }
jobs:
  docs:
    uses: project-apricot/shared-workflows/.github/workflows/docs-release.yml@v1
    permissions: { contents: read }
    secrets:
      docs_app_client_id: ${{ secrets.DOCS_APP_CLIENT_ID }}
      docs_app_private_key: ${{ secrets.DOCS_APP_PRIVATE_KEY }}

# release.yml — after a release actually reaches nuget.org
  docs:
    needs: [publish]
    if: ${{ success() }}
    uses: project-apricot/shared-workflows/.github/workflows/docs-release.yml@v1
    permissions: { contents: read }
    with:
      ref: ${{ github.event.release.tag_name }}
      version: ${{ github.event.release.tag_name }}
    secrets:
      docs_app_client_id: ${{ secrets.DOCS_APP_CLIENT_ID }}
      docs_app_private_key: ${{ secrets.DOCS_APP_PRIVATE_KEY }}
```

It pushes a fixed branch per library (`docs/<library>`) and opens a pull request, updating the
open one in place rather than stacking a second. Auto-merge is enabled on it, so it lands by
itself once the site's checks pass; merging to `main` is what triggers the deploy. Pass
`auto_merge: false` to leave it for review.

**Publication is opt-in, but the sync is not gated on it.** The site only builds libraries
listed in its `libraries.json`, and ignores a `content/docs/<slug>/` folder that has no entry.
That is deliberate: docs can be mirrored in and sit there until the manifest entry is added by
hand, which is how a library gets published for the first time.

### Credentials

`GITHUB_TOKEN` cannot write to another repository, so the workflow mints a short-lived token
from a GitHub App:

| Secret | What |
| --- | --- |
| `DOCS_APP_CLIENT_ID` | Client ID of the App installed on the docs repository |
| `DOCS_APP_PRIVATE_KEY` | Its private key |

Both are org secrets. The App needs **Contents: read & write** and **Pull requests: read &
write**, installed on `project-apricot/docs` **only** — it needs no access to the library
repositories.

**Callers must map the secrets explicitly; `secrets: inherit` does not work here.** Because
this workflow declares them under `on.workflow_call.secrets`, those names become slots that
only an explicit mapping fills. `inherit` passes the caller's secrets under their own names
(`DOCS_APP_CLIENT_ID`), which does not populate the declared `docs_app_client_id` — the
reusable workflow then sees an empty string and the token step fails with *"The 'client-id'
input must be set to a non-empty string"*. Mapping by hand also restores the `required: true`
check, so a missing secret is caught before the job starts.
