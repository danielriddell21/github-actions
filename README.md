# github-actions

Reusable workflows and composite actions shared across the repos.

## Reusable workflows

### `ci.yaml`

Test, lint, build, mutation test, tag. Every job past `test` can be switched
off, so the minimal repos call this too rather than keeping a separate copy.

| Input | Default | Notes |
| --- | --- | --- |
| `runs-on` | `ubuntu-latest` | |
| `go-version-file` | `go.mod` | |
| `test-command` | `go test -coverprofile=coverage.txt ./...` | Prefix `xvfb-run -a` when tests open a display |
| `build-command` | `go build ./...` | |
| `ebiten` | `false` | Pulls in `setup-ebiten` for every job |
| `raylib` | `false` | Pulls in `setup-raylib` for every job |
| `build-tags` | *(none)* | Build tags applied to `go vet` |
| `vet` | `false` | Adds `go vet ./...` to the test job |
| `coverage` | `true` | Codecov upload |
| `build` / `lint` / `mutate` / `tag` | `true` | Job toggles |
| `pre` | `false` | Tag a release candidate (`-rc.N`); see [Releasing](#releasing) |
| `golangci-config` | `.golangci.yml` | |
| `golangci-version` | `v2.13.2` | Pinned; see below |
| `gremlins-config` | `.gremlins.yaml` | |
| `gremlins-version` | `v0.6.0` | Pinned |
| `letsgo-version` | `latest` | Version the tag job runs |
| `release-branch` | `trunk` | Gates `mutate` and `tag` |

Secrets: `codecov-token` (when `coverage`), `tag-token` (when `tag`).

The tag token is a PAT rather than `GITHUB_TOKEN` on purpose — a tag pushed
with `GITHUB_TOKEN` does not trigger the release workflow. The tag job checks
out and pushes with it, so it needs write access to contents.

Tool versions are pinned rather than tracking `latest`. An unpinned linter
turns a branch red for a commit nobody made, which is not a hypothetical: a
golangci-lint release changed what gofumpt accepts and reddened a repo whose
last commit was days old.

Tagging is `letsgo tag --yes --warranted`, not a third-party tag action. It
reads the same conventional commits and, where the module has an importable
surface, also diffs the exported API against the previous tag — so a removal
that no commit message confessed to still reaches a major. `--warranted` means
a push with nothing but `docs:` and `chore:` commits tags nothing.

```yaml
jobs:
  ci:
    uses: danielriddell21/github-actions/.github/workflows/ci.yaml@v2
    with:
      build-command: go build -o /dev/null ./cmd/example
    secrets:
      codecov-token: ${{ secrets.CODECOV_TOKEN }}
      tag-token: ${{ secrets.EXAMPLE_TOKEN }}
```

## Releasing

Releases do not go through this repository. Each letsgo repository calls
[letsgo-action](https://github.com/danielriddell21/letsgo-action) from its own
workflows, so there is one layer between a tag and letsgo rather than two.

A repository that publishes a Homebrew formula, a cask or a container image
releases the way letsgo itself does: every trunk release is a candidate, and a
person decides when one ships.

1. A push to `trunk` runs `ci.yaml` with `pre: true`. Its tag job runs
   `letsgo tag --yes --warranted --pre` and pushes the candidate tag, such as
   `v1.4.0-rc.1`, with the repository's `tag-token`. A push of only `docs:` or
   `chore:` commits tags nothing.
2. The tag runs the repository's `release.yaml`, which publishes the candidate
   as a pre-release: the GitHub release, the `@next` formula and the image's
   `next` channel. Nothing a user installs by default moves.
3. Unticking "Set as a pre-release" on the candidate in the GitHub UI fires
   `released`, which runs `promote.yaml`. `letsgo promote` rebuilds the
   candidate, refuses if it does not match, tags `v1.4.0`, and publishes the
   stable release, the formula and any cask, and the image's `latest` and
   `X.Y` tags. The candidate goes back to being a pre-release.

What is built, the tap, the image, plugins and any cask are `letsgo.mod`'s to
say; plugins pinned there are installed by the action, and a cask is the
`letsgo-cask` plugin on the `tap-files` hook with its settings in
`.letsgo/cask.mod`. A repository that publishes none of these leaves `pre` off
and has no promote.

```yaml
name: Release

on:
  push:
    tags: ["v*"]

permissions:
  contents: write

jobs:
  release:
    name: letsgo
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-tags: true
      - uses: actions/setup-go@v7
        with:
          go-version-file: go.mod
          check-latest: false
      - uses: danielriddell21/letsgo-action@v1
        with:
          tap-app-id: ${{ vars.TAP_APP_ID }}
          tap-app-private-key: ${{ secrets.TAP_APP_PRIVATE_KEY }}
```

```yaml
name: Promote

on:
  release:
    types: [released]
  workflow_dispatch:
    inputs:
      tag:
        description: Release candidate to promote
        required: true

permissions:
  contents: write

jobs:
  promote:
    name: letsgo
    if: github.event_name == 'workflow_dispatch' || contains(github.event.release.tag_name, '-')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with:
          ref: ${{ inputs.tag || github.event.release.tag_name }}
          fetch-depth: 0
      - uses: actions/setup-go@v7
        with:
          go-version-file: go.mod
          check-latest: false
      - uses: danielriddell21/letsgo-action@v1
        with:
          command: promote
          args: --yes --work dist ${{ inputs.tag || github.event.release.tag_name }}
          tap-app-id: ${{ vars.TAP_APP_ID }}
          tap-app-private-key: ${{ secrets.TAP_APP_PRIVATE_KEY }}
```

A repository with an image adds `packages: write` to both.

## Composite actions

Checkout and Go setup are written out inline in the workflows above rather
than wrapped in an action — they are two well-known steps, and a wrapper only
adds indirection for callers that already live in this repo.

### `actions/setup-ebiten`

The OpenGL/X11/ALSA headers Ebiten links against, plus an optional virtual
display. Worth an action because the package list is long and non-obvious,
and it is applied conditionally.

| Input | Default | Notes |
| --- | --- | --- |
| `xvfb` | `false` | Installs xvfb, for jobs that wrap their own command in `xvfb-run -a` |
| `start-display` | `false` | Installs xvfb, starts it, and exports `DISPLAY` |
| `display` | `99` | Display number used by `start-display` |

`start-display` exists because gremlins runs `go test` itself, so `xvfb-run`
cannot be wrapped around it — the display has to already be up.

```yaml
- uses: danielriddell21/github-actions/.github/actions/setup-ebiten@v2
  with:
    start-display: "true"
```

### `actions/setup-raylib`

The X11/GL headers raylib links against, plus the same optional virtual
display. Same inputs as `setup-ebiten`.

The package list differs: raylib needs `libx11-dev` and `libxkbcommon-dev`,
which Ebiten does not, and does not need `libxxf86vm-dev` or `libasound2-dev`.

Unlike Ebiten — which the apps put behind a build tag, so an untagged build
still compiles — raylib is imported unconditionally and `-tags x11` selects
its backend. Every Go invocation needs the tag, which is why `build-tags`
exists for the one command the caller cannot supply whole: `go vet`.

```yaml
- uses: danielriddell21/github-actions/.github/actions/setup-raylib@v2
  with:
    xvfb: "true"
```
