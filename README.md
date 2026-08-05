# github-actions

Reusable workflows and composite actions shared across the repos.

Everything is pinned by callers to `@v1`. That tag is a moving major — push to
`trunk`, then move it:

```sh
git tag -f v1 && git push -f origin v1
```

Callers pick up the change on their next run, so treat a push to `v1` as a
change to every repo's CI at once.

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
| `vet` | `false` | Adds `go vet ./...` to the test job |
| `coverage` | `true` | Codecov upload |
| `build` / `lint` / `mutate` / `tag` | `true` | Job toggles |
| `golangci-config` | `.golangci.yml` | |
| `gremlins-config` | `.gremlins.yaml` | |
| `release-branch` | `trunk` | Gates `mutate` and `tag` |

Secrets: `codecov-token` (when `coverage`), `tag-token` (when `tag`).

The tag token is a PAT rather than `GITHUB_TOKEN` on purpose — a tag pushed
with `GITHUB_TOKEN` does not trigger the release workflow.

```yaml
jobs:
  ci:
    uses: danielriddell21/github-actions/.github/workflows/ci.yaml@v1
    with:
      build-command: go build -o /dev/null ./cmd/example
    secrets:
      codecov-token: ${{ secrets.CODECOV_TOKEN }}
      tag-token: ${{ secrets.EXAMPLE_TOKEN }}
```

### `release.yaml`

GoReleaser, triggered by a `v*` tag push in the calling repo.

| Input | Default | Notes |
| --- | --- | --- |
| `runs-on` | `ubuntu-latest` | `macos-latest` for cgo/Metal darwin builds |
| `go-version` | `stable` | |
| `homebrew-tap` | `true` | Mints an App token scoped to the tap |
| `tap-repository` | `homebrew-tap` | |
| `docker-login` | `false` | ghcr.io login before goreleaser |
| `qemu-buildx` | `false` | Multi-arch image builds |
| `goreleaser-args` | `release --clean` | |

Secrets: `tap-app-private-key`, `otel-auth-token`.

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
- uses: danielriddell21/github-actions/actions/setup-ebiten@v1
  with:
    start-display: "true"
```
