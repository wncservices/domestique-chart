# domestique-chart

The published Helm chart for [Domestique](https://github.com/wncservices/Domestique) — a shared
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
| `.github/workflows/ci.yml` | Lints and renders the chart against every example in `ci/`, and validates the rendered config against a real `domestique` binary |

## Releasing

Bump `version` in `charts/domestique/Chart.yaml` — chart-releaser skips the release entirely if
the version hasn't changed, so this is the only thing that actually triggers a publish. `appVersion`
tracks the application release this chart deploys by default (`image.tag` falls back to it); bump
that too when pointing at a new Domestique release, but it alone does not trigger anything.
