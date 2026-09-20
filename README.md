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
| `golangci-config` | `.golangci.yml` | |
| `golangci-version` | `v2.13.2` | Pinned; see below |
| `gremlins-config` | `.gremlins.yaml` | |
| `gremlins-version` | `v0.6.0` | Pinned |
| `letsgo-version` | `v0.8.0` | Version the tag job runs |
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

### `release.yaml`

[letsgo](https://github.com/danielriddell21/letsgo), triggered by a `v*` tag
push in the calling repo.

| Input | Default | Notes |
| --- | --- | --- |
| `runs-on` | `ubuntu-latest` | Every target cross-compiles; no macOS runner needed |
| `go-version-file` | `go.mod` | The compiler is a build input, so it is pinned by file |
| `letsgo-version` | `v0.8.0` | |
| `plugins` | *(none)* | e.g. `letsgo-env letsgo-multi` |
| `plugins-version` | `v0.2.0` | letsgo-plugins release to install from |
| `homebrew-tap` | `true` | Mints an App token scoped to the tap |
| `tap-repository` | `homebrew-tap` | |
| `cask-name` | *(none)* | Set it to write a cask; empty means none |
| `cask-variant` | *(none)* | Variant whose archives the cask installs |
| `cask-desc` / `cask-license` / `cask-caveats` | / `MIT` / | Cask metadata |

Secrets: `tap-app-private-key`, `otel-auth-token`.

What a release builds is the calling repository's `letsgo.mod` rather than an
input here. The target matrix, the archive contents, the tap and the container
image are all build inputs, and a build input belongs in the repository it
describes, pinned by the same commit as the source.

That includes whether a release is a pre-release: `letsgo.mod` says
`release prerelease=true` and letsgo publishes it that way, rather than the
workflow patching it afterwards. Promotion is still manual, and still what
`promote.yaml` waits for — marking a release as the full release fires the
`released` event. Nothing is retagged or deployed until you make that call.

#### Casks

letsgo writes formulas, not casks, and a variant is where the two part company:
the headless build belongs in a formula and the windowed one in a cask. Setting
`cask-name` adds a second job that reads the manifest the release just
published and writes the cask into the tap.

```yaml
jobs:
  release:
    permissions:
      contents: write
    uses: danielriddell21/github-actions/.github/workflows/release.yaml@v2
    with:
      cask-name: gambit
      cask-variant: gui
      cask-desc: Watch two chess agents play in a native macOS window
      cask-caveats: The board opens a window and is macOS-only; elsewhere install the formula.
    secrets:
      tap-app-private-key: ${{ secrets.TAP_APP_PRIVATE_KEY }}
```

`vars.TAP_APP_ID` and `vars.OTEL_ENDPOINT` resolve against the *calling*
repository, so they are read directly and are not inputs.

### `promote.yaml`

Retag the released image as `latest`, then pin the version into the manifests
repo. Triggered by a `released` release event.

| Input | Default |
| --- | --- |
| `image` | calling repo's name |
| `registry` | `ghcr.io` |
| `manifests-repo` | `danielriddell21/riddellious-dev` |
| `manifests-ref` | `trunk` |
| `manifests-path` | `manifests/` |

Secrets: `manifests-token`.

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
