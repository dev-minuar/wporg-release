# wporg-release

Reusable GitHub workflows that publish a WordPress plugin to WordPress.org. They wrap the 10up SVN actions and add checks before anything reaches SVN.

## Workflows

1. `deploy.yml` checks the plugin, then commits `trunk` and a tag to WordPress.org SVN and waits until WordPress.org serves the new version.
2. `assets.yml` pushes `readme.txt` and the `.wordpress-org` folder (banners, icons, screenshots) to SVN without a release.

Both run as `@v1`.

## Set up a plugin repository

Copy these two files into the plugin repository.

`.github/workflows/wporg-deploy.yml`:

```yaml
name: WordPress.org deploy

on:
  push:
    tags:
      - 'v*'
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to deploy (e.g. 1.1.0)'
        required: true
      dry-run:
        description: 'Dry run (no SVN commit)'
        type: boolean
        default: true

jobs:
  deploy:
    uses: dev-minuar/wporg-release/.github/workflows/deploy.yml@v1
    with:
      version: ${{ inputs.version || '' }}
      dry-run: ${{ github.event_name == 'workflow_dispatch' && inputs.dry-run || false }}
    secrets: inherit
```

`.github/workflows/wporg-assets.yml`:

```yaml
name: WordPress.org readme and assets

on:
  workflow_dispatch:

jobs:
  assets:
    uses: dev-minuar/wporg-release/.github/workflows/assets.yml@v1
    secrets: inherit
```

`secrets: inherit` across repositories works only when caller and this repository belong to the same organization. Otherwise pass `SVN_USERNAME` and `SVN_PASSWORD` under `secrets:` by name.

Put banners, icons and screenshots in `.wordpress-org/` and list `.wordpress-org`, `.wporg-dist` and `.wp-env.json` in `.distignore`.

Warning: rsync excludes match at any depth. Anchor entries that apply only to the repository root with a leading `/` (`/vendor`, `/node_modules`, `/docs`). An unanchored `vendor` also removes `assets/vendor/` from the package and deletes it from SVN trunk.

## One-time secret setup

The SVN password is the WordPress.org SVN password, not the login password. Set both secrets once per plugin repository. The password is piped from the macOS keychain, so it never appears on a command line:

```bash
gh secret set SVN_USERNAME --repo OWNER/PLUGIN --body minuar
security find-generic-password -a minuar -s '<https://plugins.svn.wordpress.org:443> Use your WordPress.org login' -w | gh secret set SVN_PASSWORD --repo OWNER/PLUGIN
```

## Release

Preconditions:

1. The plugin repository has a `.distignore` in its root. The deploy fails early without it, because 10up would otherwise build the package from `.gitattributes` and Plugin Check would check different files than ship.
2. Do not run the assets workflow between the version bump commit and the tag push. The assets workflow now fails when `tags/<Stable tag>` does not exist on WordPress.org, but run the deploy first anyway.
3. The repository has the secrets `SVN_USERNAME` and `SVN_PASSWORD`. The deploy job needs them even for a dry run.

Bump `Version:` in the main plugin file and `Stable tag:` in `readme.txt`, commit, then:

```bash
git tag vX.Y.Z && git push origin vX.Y.Z
```

The tag name without the leading `v` is the version. For a manual run, use the `version` input as it is (no `v`).

## Update readme and assets only

Open the Actions tab, choose "WordPress.org readme and assets", then Run workflow. From the terminal: `gh workflow run wporg-assets.yml`.

Warning: this publishes the readme and assets from `main` to WordPress.org immediately.

## Checks in deploy.yml

1. Version match. The tag version, the `Version:` header of the main plugin file, and `Stable tag:` in `readme.txt` must be equal. A mismatch fails the job before any SVN step.
2. PHP syntax. Every `.php` file outside `vendor` and `node_modules` must pass `php -l`.
3. Readme validator. The WordPress.org online validator must report no Fatal or Warnings lines. A private repository skips this check with a warning.
4. Plugin Check. The files that `.distignore` keeps run through WordPress Plugin Check. Warnings are ignored; errors fail the job. Set `plugin-check: false` to skip.
5. `.distignore`. The file must exist. The job fails early without it.
6. Dry run. With `dry-run: true` every step runs except the SVN commit and the wait for WordPress.org. The 10up step still needs `SVN_USERNAME` and `SVN_PASSWORD` and fails without them, so a dry run without secrets passes the checks job and fails the deploy job.

`deploy.yml` has two jobs. Job `checks` (steps 1 to 5 and Plugin Check) holds no secrets, because it runs third-party code. Job `deploy` runs on a fresh runner after `checks` passes, and only the 10up step receives the secrets.

`assets.yml` reads `Stable tag:` from `readme.txt` and fails unless `tags/<Stable tag>` exists on WordPress.org.

## Action pins

Every action is pinned to a full commit SHA with its version in a trailing comment. To bump a pin, resolve the new tag with `gh api repos/OWNER/REPO/git/ref/tags/TAG --jq .object.sha` (for an annotated tag, read the commit through `git/tags/SHA`), replace the SHA and the comment, run `actionlint`, then move the `v1` tag.

## Inputs

`deploy.yml`: `version` (default: tag without `v`), `slug` (default: repository name), `dry-run` (default false), `plugin-check` (default true). Secrets `SVN_USERNAME` and `SVN_PASSWORD` are declared optional, but the deploy job fails without them, also on a dry run.

`assets.yml`: `slug` (default: repository name). Both secrets are required.

## License

MIT.
