# e2e-mirror

The tier-2 canary for `ocx-mirror`. One repository, one job: prove end to end,
against real infrastructure, the things an offline harness structurally cannot.

Today that is **publish-side signing**. The spec in `signing/` mirrors
`pycowsay` from PyPI into `dev.ocx.sh/ocx/e2e-signing` and signs it keyless
against Sigstore using the GitHub Actions workflow's own OIDC identity.

## Why a separate repository

`ocx-mirror`'s own acceptance suite runs against a local registry and a
compose Sigstore stack. Four things never appear there:

- a real Fulcio certificate exchange driven by a real OIDC token;
- the `id-token: write` permission actually being granted to the job that
  needs it, in a workflow the renderer wrote rather than a human;
- `ocx package verify` accepting a signature against the workflow identity
  that produced it;
- the index above the platform manifests carrying a signature — a referrer
  is filed against the subject digest, and that leg broke twice during
  development without any single-manifest test noticing.

## Layout

| Path | Purpose |
|---|---|
| `ocx.toml` / `ocx.lock` | toolchain pins — `ocx-mirror` comes from the `dev.ocx.sh` dev channel, not a release |
| `signing/mirror.yml` | the canary spec, including the `sign:` block |
| `signing/tests/` | starlark smoke tests run on every platform leg |
| `.github/workflows/` | generated — `ocx-mirror package pipeline generate ci`; do not hand-edit |

## Pinning a dev build

`ocx.toml` deliberately pins `mirror` to a `dev.ocx.sh` build rather than a
release, which is how an unreleased `ocx-mirror` branch gets exercised against
real infrastructure before it merges:

```bash
gh workflow run "Deploy Dev" --repo ocx-sh/ocx-mirror --ref <branch>
# then put the published tag in ocx.toml and re-run `ocx lock`
```

Everything published here goes to `dev.ocx.sh`. Nothing is published to GHCR.

## Which `ocx` actually runs

`setup-ocx`'s `version:` input — baked as `0.6.0` by the renderer — installs
only the **bootstrap** `ocx` that performs project activation. It then
activates `ocx.toml`, so the `ocx` and `ocx-mirror` the push step invokes both
come from the pins above. The generated push step calls `ocx-mirror` directly
rather than through `ocx exec` precisely to keep it that way.

That is what makes an unreleased `ocx` usable here: keyless signing needs
`--fulcio-url` / `--rekor-url` on `ocx package push --sign`, which landed after
0.6.0, and pinning a `dev.ocx.sh` cli supplies them with no release cut.

