# .github

# Reusable workflows

Shared GitHub Actions workflows for Bernskiold repositories. Call them from a
thin workflow in your repo with
`uses: bernskiold/.github/.github/workflows/<file>@main`.

| Workflow | Purpose |
| --- | --- |
| [`wordpress-plugin-tests.yml`](.github/workflows/wordpress-plugin-tests.yml) | PHPUnit against real WordPress over a PHP x WP matrix, optional PHP/JS lint, committed-build check, Jest and Playwright e2e |
| [`wordpress-plugin-release.yml`](.github/workflows/wordpress-plugin-release.yml) | Manually triggered release: bump versions, changelog, build, one commit + tag, publish the release with the plugin ZIP, optional Satis rebuild |
| [`satis-notify.yml`](.github/workflows/satis-notify.yml) | Ask packages.bernskiold.com to rebuild its Satis index for this repo (any Composer package) |
| [`dependabot-auto-merge.yml`](.github/workflows/dependabot-auto-merge.yml) | Enable auto-merge (squash) on Dependabot patch and minor PRs |
| [`update-changelog.yml`](.github/workflows/update-changelog.yml) | Prepend the release notes to `CHANGELOG.md` (non-WordPress repos; the plugin release workflow already does it) |

## `wordpress-plugin-tests.yml`

| Input | Type | Default | Description |
| --- | --- | --- | --- |
| `php-versions` | string (JSON array) | `'["8.2","8.3","8.4","8.5"]'` | PHP versions in the PHPUnit matrix |
| `wp-versions` | string (JSON array) | `'["7.0","latest"]'` | WordPress versions in the PHPUnit matrix |
| `php-extensions` | string | `'mysqli'` | PHP extensions for the PHPUnit job, in shivammathur/setup-php `extensions` syntax, e.g. `'mysqli, imagick'`. Keep `mysqli`, since the WP test suite needs it |
| `node-version` | string | `'26'` | Node.js version |
| `build` | boolean | `true` | Run `npm ci && npm run build` before the PHP tests (skipped if there is no `package.json`) |
| `js-unit` | boolean | `false` | Run `npm run test:unit` |
| `e2e` | boolean | `false` | Start wp-env (`npm run env:start`) and run Playwright (`npm run test:e2e`); the report in `artifacts/` is uploaded |
| `php-lint` | boolean | `false` | Run `composer lint` (phpcs) |
| `js-lint` | boolean | `false` | Run `npm run lint:js` and `npm run lint:css`, each only if the script exists |
| `check-dist` | boolean | `false` | Run `npm run build` and fail if `git status --porcelain` shows changes, i.e. the committed build output does not match the sources |
| `working-directory` | string | `'.'` | Plugin directory relative to the repo root. Every npm, composer and PHPUnit step runs there, and the npm cache, composer install and Playwright report paths follow it. Use it for a plugin inside a site repo, e.g. `wp-content/plugins/ehs-courses` |

The plugin must provide `bin/install-wp-tests.sh [wp-version] [db-name] [db-user] [db-pass] [db-host]`
(as in bm-concierge / bm-block-library) that prints `KEY=value` lines such as
`WP_PHPUNIT__DIR=...`, and `composer test` running PHPUnit. A MySQL 8 service is
provided. The `php-lint` and e2e jobs use the first entry of `php-versions`.

For a plugin that lives inside a site repo, point the workflow at it:

```yaml
jobs:
    tests:
        uses: bernskiold/.github/.github/workflows/wordpress-plugin-tests.yml@main
        with:
            working-directory: wp-content/plugins/ehs-courses
```

```yaml
# .github/workflows/tests.yml
name: Tests

on:
    push:
        branches: [main]
    pull_request:

permissions:
    contents: read

concurrency:
    group: tests-${{ github.ref }}
    cancel-in-progress: true

jobs:
    tests:
        uses: bernskiold/.github/.github/workflows/wordpress-plugin-tests.yml@main
        with:
            php-versions: '["8.2","8.3","8.4","8.5"]'
            wp-versions: '["7.0","latest"]'
            js-unit: true
            e2e: true
```

## `wordpress-plugin-release.yml`

Releases the tip of the default branch. Callers trigger it manually with
`workflow_dispatch`, so the tag is created by the workflow, exactly once,
on the final release commit.

| Input | Type | Default | Description |
| --- | --- | --- | --- |
| `version` | string | required | Version to release, semver without a leading `v` (`1.2.3`, `2.0.0-beta.1`) |
| `notes` | string | `''` | Release notes. Ignored when a draft release for the tag exists; empty means GitHub's generated notes |
| `prerelease` | boolean | `false` | Publish as a pre-release; `CHANGELOG.md` is not updated |
| `build` | boolean | `true` | Run `npm ci --ignore-scripts && npm run build` before committing |
| `node-version` | string | `'26'` | Node.js version for the build |
| `main-file` | string | `''` | Main plugin file with the `Version:` header; empty means `<repo-name>.php` |
| `version-files` | string | `'src/Plugin.php'` | Newline-separated files in which `version = '...'` is bumped (missing files are skipped with a warning; pass `''` for none) |
| `changelog` | boolean | `true` | Prepend the release notes to `CHANGELOG.md` (if the file exists) in the release commit; skipped for pre-releases |
| `composer` | boolean | `false` | Run `composer install --no-dev --optimize-autoloader` after the release commit and ship `vendor/` in the ZIP |
| `satis` | boolean | `false` | Ask packages.bernskiold.com to rebuild its Satis index after the release (needs `secrets: inherit`) |

What it does, in order:

1. Checks out the default branch (`main` or `master`) with full history.
2. Fails if `version` isn't valid semver, if the tag `x.y.z` or `vx.y.z` already
   exists, or if `version` isn't greater than the latest published release.
3. Picks the release notes, in this order:
   - the body of a **draft** release whose tag is `x.y.z` (or `vx.y.z`)
   - otherwise the `notes` input
   - otherwise GitHub's generated notes since the previous release
4. Bumps `package.json`, the `Version:` header and the `version-files`. Prepends
   the notes to `CHANGELOG.md` (skipped for pre-releases or if the file is
   missing), builds, and makes one commit, "Release x.y.z".
5. Creates an annotated tag `x.y.z` on that commit and pushes the branch and
   the tag together with `git push --atomic`. If the branch moved in the
   meantime, nothing is pushed and the run fails; run it again.
6. Builds `<repo-name>.zip`. This is a `git archive` of the release commit, so
   `.gitattributes` `export-ignore` applies. With `composer: true` the
   production `vendor/` is added, and a `.distignore` (rsync patterns, as used
   by `wp dist-archive`) is applied if present. `vendor/` is installed after
   the commit, so it is never committed.
7. Publishes the release: the draft is pointed at the tag and published, or a
   new release is created with the notes. The ZIP is attached either way.
8. With `satis: true`, runs `satis-notify.yml` as a final job.

Only one release per repository runs at a time.

**Why the tag is never moved.** Packagist (and Satis) index a tag the first
time they see it. They ignore, or refuse, a tag that is later force-moved to
another commit. The old flow published a release, which created the tag, and
then committed the version bump and moved the tag. Packagist kept serving the
pre-bump commit. Now the tag is only ever created on the final commit.

**Why Satis is notified from inside the workflow.** The tag push and the
release are made with `GITHUB_TOKEN`, which never triggers other workflows in
the repo. A separate `notify-satis.yml` listening for tags or releases would
not fire. Use `satis: true` instead.

```yaml
# .github/workflows/release.yml
name: Release

on:
    workflow_dispatch:
        inputs:
            version:
                description: Version to release, without a leading "v" (e.g. 1.2.3)
                required: true
                type: string
            notes:
                description: Release notes (optional; a draft release's notes or generated notes are used otherwise)
                required: false
                type: string
            prerelease:
                description: Pre-release
                required: false
                type: boolean
                default: false

permissions:
    contents: write

jobs:
    release:
        uses: bernskiold/.github/.github/workflows/wordpress-plugin-release.yml@main
        with:
            version: ${{ inputs.version }}
            notes: ${{ inputs.notes }}
            prerelease: ${{ inputs.prerelease }}
            version-files: |
                src/Plugin.php
            satis: true
        secrets: inherit
```

### How to release

1. Optional: write the notes ahead of time. Create a **draft** release (Releases →
   Draft a new release) with the tag `1.2.3` typed in but not published. A
   draft does not create the tag.
2. Run the workflow from **Actions → Release → Run workflow**, enter the
   version, and optionally paste notes. From a terminal:

   ```bash
   gh workflow run release.yml -f version=1.2.3
   gh workflow run release.yml -f version=1.3.0-beta.1 -f prerelease=true
   gh workflow run release.yml -f version=1.2.3 -f notes="Fixes the settings page."
   ```

Do not publish releases or push tags by hand. Doing that brings back the
problem described above.

## `satis-notify.yml`

| Input | Type | Default | Description |
| --- | --- | --- | --- |
| `repository-url` | string | `''` | URL as listed in `satis.json`; empty means `https://github.com/<owner>/<repo>` |
| `satis-repository` | string | `'bernskiold/packages.bernskiold.com'` | Repository that builds the Satis index |

Needs the `SATIS_DISPATCH_TOKEN` org secret, so call it with `secrets: inherit`.
When triggered by `workflow_run` it only dispatches if that run succeeded.
Concurrent triggers for the same repository collapse into one dispatch.

WordPress plugins using `wordpress-plugin-release.yml` don't need a separate
caller: set `satis: true` there. For other Composer packages that tag
releases by hand, such as Laravel packages:

```yaml
# .github/workflows/notify-satis.yml
name: Notify Satis

on:
    push:
        tags: ["*"]
    release:
        types: [published]
    workflow_dispatch:

jobs:
    notify:
        uses: bernskiold/.github/.github/workflows/satis-notify.yml@main
        secrets: inherit
```

## `dependabot-auto-merge.yml`

No inputs. Enables auto-merge (squash) on Dependabot patch and minor updates;
majors are left for review. Requires "Allow auto-merge" in the repo settings.

```yaml
# .github/workflows/dependabot-auto-merge.yml
name: Dependabot Auto-Merge

on: pull_request_target

jobs:
    auto-merge:
        uses: bernskiold/.github/.github/workflows/dependabot-auto-merge.yml@main
        permissions:
            contents: write
            pull-requests: write
```

## `update-changelog.yml`

No inputs. Prepends the release notes to `CHANGELOG.md` on the repo's default branch (`main` or `master`).

**WordPress plugins using `wordpress-plugin-release.yml` don't need this.** The
release workflow updates the changelog in its own "Prepare release" commit, so
there is a single push to the default branch. Running both would make two
workflows race to push. Use this for repos without that release workflow, such
as Laravel packages.

```yaml
# .github/workflows/update-changelog.yml
name: Update Changelog

on:
    release:
        types: [released]

jobs:
    update:
        uses: bernskiold/.github/.github/workflows/update-changelog.yml@main
        permissions:
            contents: write
```
