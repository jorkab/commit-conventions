# commit-conventions

Shared source of truth for Conventional Commit types, consumed by both [commitlint](https://commitlint.js.org/) and [release-it](https://github.com/release-it/release-it)'s [conventional-changelog plugin](https://github.com/release-it/conventional-changelog).

## Why

`commitlint` validates that commit messages use an allowed `type`, and `release-it` groups commits by `type` into changelog sections. Without a shared source, those two type lists drift apart: a `type` gets added to lint rules but forgotten in the changelog preset (or vice versa), and commits silently disappear from — or get rejected by — one tool but not the other.

This package defines the commit types once and derives both configs from it, so they can never disagree.

## Installation

```bash
npm install --save-dev commit-conventions
```

## Usage

The package exports two ready-to-use configs:

```js
const { commitlint, releaseIt } = require('commit-conventions');
```

### commitlint

```js
// commitlint.config.js
module.exports = require('commit-conventions').commitlint;
```

### release-it

```js
// release-it.config.js
module.exports = require('commit-conventions').releaseIt;
```

> Each tool's `--config` flag expects the file it points to export a single config object, so `commitlint` and `release-it` need separate files even though both re-export from this package.

## Overriding configuration

`commitlint` and `releaseIt` are plain JS objects, not factory functions, so a consumer that needs a different rule, section, or option doesn't fork the package — it merges its own values on top of the exported config in its own config file.

### commitlint

```js
// commitlint.config.js
const { commitlint } = require('commit-conventions');

module.exports = {
    ...commitlint,
    rules: {
        ...commitlint.rules,
        'subject-case': [2, 'always', 'lower-case'],
    },
};
```

### release-it

`releaseIt` has nested objects (`git`, `github`, `npm`, `plugins`), so a shallow spread only replaces top-level keys wholesale. Spread each nested key you're touching too, or the rest of that section is lost:

```js
// release-it.config.js
const { releaseIt } = require('commit-conventions');

module.exports = {
    ...releaseIt,
    git: {
        ...releaseIt.git,
        requireBranch: 'release/*',
    },
};
```

To add a commit type without losing the shared list, spread the existing `types` array from the `conventional-changelog` plugin preset:

```js
// release-it.config.js
const { releaseIt } = require('commit-conventions');

module.exports = {
    ...releaseIt,
    plugins: {
        ...releaseIt.plugins,
        '@release-it/conventional-changelog': {
            ...releaseIt.plugins['@release-it/conventional-changelog'],
            preset: {
                ...releaseIt.plugins['@release-it/conventional-changelog'].preset,
                types: [
                    ...releaseIt.plugins['@release-it/conventional-changelog'].preset.types,
                    { type: 'security', section: 'Security' },
                ],
            },
        },
    },
};
```

> Adding a type here only affects the changelog. If commits of that type also need to pass `commitlint`, add it to the `commitlint` override above too — the two lists don't sync automatically once you override them.

## Commit types

| Type | Changelog section | In changelog |
|---|---|---|
| `feat` | Features | yes |
| `fix` | Bug Fixes | yes |
| `perf` | Performance Improvements | yes |
| `revert` | Reverts | yes |
| `docs` | Documentation | yes |
| `refactor` | Code Refactoring | yes |
| `style` | — | hidden |
| `chore` | — | hidden |
| `test` | — | hidden |
| `build` | — | hidden |
| `ci` | — | hidden |

Hidden types are still valid commits — `commitlint` accepts them — they're just omitted from the generated `CHANGELOG.md`.

## Used with gha-shared

This package is meant to be paired with [jorkab/gha-shared](https://github.com/jorkab/gha-shared), which hosts the reusable GitHub Actions workflows that run `commitlint` and `release-it` in caller repos.

```yaml
# .github/workflows/commitlint.yml — on pull requests
name: Lint Commit Messages

on:
  pull_request:

permissions:
  contents: read

jobs:
  commitlint:
    uses: jorkab/gha-shared/.github/workflows/commitlint.yml@v1
    with:
      config-file: commitlint.config.js
```

```yaml
# .github/workflows/release.yml — on push to main
name: Release it

on:
  push:
    branches:
      - main

permissions:
  contents: write
  issues: write
  pull-requests: write

jobs:
  release:
    uses: jorkab/gha-shared/.github/workflows/release-it.yml@v1
    with:
      config-file: release-it.config.js
    secrets:
      release-token: ${{ secrets.MY_RELEASE_PLEASE_TOKEN }}
```

Both `gha-shared` workflows default `config-file` to `release.config.js`; since `commitlint` and `release-it` each need their own file (see [Usage](#usage)), point each workflow at its matching config as shown above. The caller repo also needs a `.nvmrc` and a committed `package-lock.json` for `npm ci` to succeed.
