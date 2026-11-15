# domestique-chart

The published Helm chart for [Domestique](https://github.com/wncservices/domestique) — a shared
cycling route library that syncs to Garmin and Wahoo head units. Free software under the
[GNU AGPL-3.0](LICENSE), same as the app it deploys.

Split out of the app repo into its own repo so the chart can be versioned, released and consumed
independently of the application's own release cycle — see
[`charts/domestique/README.md`](charts/domestique/README.md) for install instructions, required
values, and everything else about actually deploying it.

```bash
helm repo add domestique https://wncservices.github.io/domestique-chart
helm repo update
```

## Layout

| Path | What |
|---|---|
| `charts/domestique/` | The chart itself — `Chart.yaml`, `values.yaml`, `templates/`, worked examples under `ci/` |
| `.github/workflows/chart-release.yml` | Publishes to GitHub Pages via `chart-releaser-action` on every push to `main` that bumps `charts/domestique/Chart.yaml`'s `version` |
| `.github/workflows/app-release.yml` | Every 15 minutes (and on demand), points `appVersion` at the latest Domestique release, bumps the chart patch version, and starts `chart-release.yml` |
| `.github/workflows/ci.yml` | Lints and renders the chart against every example in `ci/`, and validates the rendered config against a real `domestique` binary |

## Releasing

Bump `version` in `charts/domestique/Chart.yaml` — chart-releaser skips the release entirely if
the version hasn't changed, so this is the only thing that actually triggers a publish. `appVersion`
tracks the application release this chart deploys by default (`image.tag` falls back to it); on
its own it does not trigger anything.

A new Domestique release needs no hand bump: `app-release.yml` notices it within about 15 minutes,
sets `appVersion`, bumps the chart's patch version and publishes the chart, which is what Kargo
picks up. To ship it sooner, run that workflow by hand (`gh workflow run app-release.yml`). Chart
changes of your own still bump `version` by hand, as above; the workflow only ever moves
`appVersion` forward.
