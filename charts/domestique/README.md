# Domestique Helm chart

Deploys [Domestique](https://github.com/wncservices/Domestique) — a shared
cycling route library that syncs to Garmin and Wahoo head units.

## Install

```bash
helm repo add domestique https://wncservices.github.io/Domestique-chart
helm repo update
helm install domestique domestique/domestique \
  --namespace domestique --create-namespace
```

**PostgreSQL is required** — see below; the chart refuses to render without
one. With nothing else set you get a single pod, no ingress and **no
authentication**. Port-forward to look at it:

```bash
kubectl -n domestique port-forward svc/domestique 8080:80
```

## Two things to set before exposing it

**Authentication is off by default, which means every visitor is an admin** —
they can upload, delete and push routes. Domestique authenticates nobody
itself; it reads the identity a reverse proxy passes down. Behind Traefik with
an Authelia forwardAuth middleware:

```yaml
config:
  auth:
    mode: proxy
    trusted_proxies:
      - 10.0.0.0/8          # your pod CIDR
    roles:
      admin: [route-admins]
      rider: [riders]
      viewer: [guests]
```

`mode: proxy` makes the app trust the `Remote-User` header, so **the Service
must not be reachable except through that proxy**. `trusted_proxies` narrows it
further; leave it empty only for a ClusterIP-only Service.

Alternatively, `mode: oidc` has the app verify signed tokens itself against
an OIDC issuer (Auth0, Keycloak, Zitadel, ...) rather than trusting a header
— the mode for a Service that faces the public directly:

```yaml
config:
  auth:
    mode: oidc
    oidc:
      issuer: https://your-tenant.example.com/
      client_id: domestique
      redirect_url: https://app.example.com/sso/callback
      scopes: [openid, profile, email]
      groups_claim: groups
    roles:
      admin: [domestique-admins]
      rider: [cyclists]
encryptionKey:
  existingSecret: domestique       # DOMESTIQUE_ENCRYPTION_KEY; required for this mode
```

The client secret comes from `DOMESTIQUE_OIDC_CLIENT_SECRET` via `envFrom`,
never from `config`. An optional, separate `DOMESTIQUE_AUTH0_MGMT_CLIENT_ID` /
`_SECRET` pair (an Auth0 Management API M2M app, narrowly scoped) enables the
admin People page for inviting riders and managing access — without it the
page is simply unavailable, not broken.

## Exposing it

**One ingress resource, deliberately** — a plain `networking.k8s.io/v1
Ingress`, so the chart works with whatever controller a cluster already has
rather than assuming one:

```yaml
ingress:
  enabled: true
  className: traefik          # your ingress controller's class
  hosts:
    - host: domestique.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: domestique-tls
      hosts: [domestique.example.com]
```

**Running Traefik?** No separate CRD needed — Traefik is a supported
controller for a standard `Ingress`, via annotations instead of its own
`IngressRoute`:

```yaml
ingress:
  enabled: true
  className: traefik
  annotations:
    traefik.ingress.kubernetes.io/router.tls: "true"
    traefik.ingress.kubernetes.io/router.tls.certresolver: cloudflare   # whatever resolver you already have
    # traefik.ingress.kubernetes.io/router.middlewares: traefik-internal@kubernetescrd
  hosts:
    - host: domestique.example.com
      paths: [{path: /, pathType: Prefix}]
  tls:
    - secretName: domestique-tls
      hosts: [domestique.example.com]
```

Any other controller works the same way, with its own annotations in
`ingress.annotations` — cert-manager's `cert-manager.io/cluster-issuer`,
for instance. `ci/full-values.yaml` has the Traefik example above rendered
in full, and CI renders it on every change.

## PostgreSQL, and no volume

Routes, sync state, linked head units and Komoot sign-ins are all rows in one
database, so the pod keeps nothing of its own. There is no `persistence` block
any more and no PVC: with PostgreSQL there was nothing left to put on a volume,
and an empty one that silently did nothing was worse than none.

The usual case is a [CloudNativePG](https://cloudnative-pg.io) cluster in the
same namespace:

```yaml
postgresql:
  cluster: domestique-db
```

The operator publishes `Secret/domestique-db-app` holding a ready-made
connection string that already points at the read-write service. The chart
reads it directly, so there is no copy of the password anywhere and nothing to
change when the operator rotates it or the cluster fails over.

**No CNPG cluster yet? The chart can create one for you:**

```yaml
postgresql:
  enabled: true
  cluster: domestique-db      # still required — names the Cluster this creates
  instances: 1
  storage:
    size: 5Gi
```

This still needs the **CNPG operator itself already installed** in the
target cluster — it creates a `Cluster` custom resource for the operator to
reconcile, it does not install the operator. `helm lint`/`helm template`
render this fine either way; a missing operator only surfaces as a real
`helm install` failure, since the CRD comes from the operator, not this
chart. `postgresql.cluster` does double duty on purpose: it is both the name
of the `Cluster` this creates and the name the chart reads the resulting
`<cluster>-app` Secret from, so there is exactly one field to get right, not
two that have to agree.

Any other PostgreSQL works too:

```yaml
postgresql:
  existingSecret: domestique-db
  secretKey: uri
```

**SQLite is not a deployment option.** It still runs a laptop (`just up`,
`domestique serve`), but as a deployment it means one replica pinned to one
node holding the only copy of the library on a disk nothing backs up. The
chart used to offer it and defaulted to it, which is the wrong default to
reach for by accident.

## Komoot sign-in

Riders connect their own Komoot account from the UI. The session is stored
encrypted, so it needs a key:

```bash
domestique keygen
```

Put it in a Secret and name it:

```yaml
encryptionKey:
  existingSecret: domestique
  secretKey: DOMESTIQUE_ENCRYPTION_KEY
```

Without it the sign-in form is not offered and everything else works as
normal. Replacing the key invalidates every stored sign-in; riders sign in
again.

## Other credentials

A DSN carries a password, so it
comes from a Secret rather than values:

```yaml
config:
  source: {}          # no dsn here
envFrom:
  - secretRef:
      name: domestique      # must contain DOMESTIQUE_SOURCE_DSN
```

Create that Secret however you manage secrets — an ExternalSecret from Vault,
for instance. Keys the app reads:

| Key | For |
|---|---|
| `KOMOOT_EMAIL` | one shared Komoot account, as an alternative to riders signing in themselves |
| `KOMOOT_PASSWORD` | |
| `GARMIN_OAUTH_CONSUMER_KEY` | the OAuth1 consumer Garmin Connect's own clients use — one pair for the whole deployment. **Optional**: an admin can paste the pair into Settings instead, stored encrypted, which takes precedence. Supply it here to keep it in Vault rather than in the database. Not shipped with this chart or its source |
| `GARMIN_OAUTH_CONSUMER_SECRET` | |

`DOMESTIQUE_SOURCE_DSN` and `DOMESTIQUE_ENCRYPTION_KEY` are set by the chart
from the Secrets named above; they do not go in `envFrom`.

## Backups

**Off by default.** Domestique has never taken its own backups, on purpose —
this is a second, opt-in system, not something the app relies on existing.
`backup.enabled: true` adds a `CronJob` running `pg_dump` against the same
database `postgresql`/`envFrom` above already point at, on `backup.schedule`
(default `0 3 * * *`).

Pick exactly one destination — the chart refuses to render with neither, or
both:

```yaml
backup:
  enabled: true
  destination:
    pvc:
      claimName: domestique-backups   # a PVC to dump into
```

or

```yaml
backup:
  enabled: true
  destination:
    # A Secret with exactly these four keys: endpoint, bucket, accessKeyId,
    # secretAccessKey. Any S3-compatible store, not AWS specifically.
    existingSecret: domestique-backup-s3
```

`backup.retention` (days, default `7`) only applies to the `pvc` path —
old dumps get pruned from the same volume on every run. The `existingSecret`
path does not prune anything; most S3-compatible stores have their own
bucket lifecycle rules, a better fit than a CronJob re-listing a bucket.

## Values

| Key | Default | What |
|---|---|---|
| `image.repository` | `ghcr.io/wncservices/domestique` | |
| `image.tag` | chart `appVersion` | |
| `replicaCount` | `1` | Leave at 1 — two replicas race on the state file |
| `config` | see `values.yaml` | Rendered into a ConfigMap as the app's config file. **No secrets** |
| `envFrom` | `[]` | Where credentials come from |
| `postgresql.cluster` | `""` | CloudNativePG cluster in this namespace. **Required**, unless `existingSecret` |
| `postgresql.existingSecret` | `""` | Any Secret holding a PostgreSQL URL; wins over `cluster` |
| `postgresql.secretKey` | `uri` | Key within that Secret |
| `postgresql.enabled` | `false` | Create the `postgresql.cluster` Cluster instead of requiring it to exist. Needs the CNPG operator already installed |
| `postgresql.instances` | `1` | Only used when `enabled: true` |
| `postgresql.storage.size` | `2Gi` | Only used when `enabled: true` |
| `postgresql.storage.storageClass` | `""` | Only used when `enabled: true`; empty uses the cluster default |
| `encryptionKey.existingSecret` | `""` | Enables Komoot sign-in from the UI |
| `backup.enabled` | `false` | Scheduled `pg_dump`. Needs exactly one of the two destinations below |
| `backup.schedule` | `0 3 * * *` | |
| `backup.destination.pvc.claimName` | `""` | Dump into this PVC, pruning anything older than `retention` |
| `backup.destination.existingSecret` | `""` | Dump to an S3-compatible store; Secret needs `endpoint`/`bucket`/`accessKeyId`/`secretAccessKey` |
| `backup.retention` | `7` | Days of dumps to keep. Only applies to the `pvc` destination |
| `ingress.enabled` | `false` | A plain `Ingress` — works with any controller, see **Exposing it** above |
| `serviceAccount.name` | release name | Vault's Kubernetes auth binds to this |
| `podDisruptionBudget.enabled` | `true` | `maxUnavailable: 1`, so node drains still work with one replica |
| `automountServiceAccountToken` | `true` | Needed if the app authenticates to Vault |
| `revisionHistoryLimit` | `3` | |

`ci/full-values.yaml` is a complete worked example: PostgreSQL, a proxy doing
authentication, Traefik ingress and Secret-backed credentials. The values in it
are placeholders.

## What this chart does not do

- **Create the database, by default.** Point `DOMESTIQUE_SOURCE_DSN` at one that exists, or opt in to `postgresql.enabled: true` if you'd rather the chart create a CNPG `Cluster` for you.
- **Create the Secret.** That is your secret manager's job.
- **Provide a ServiceMonitor.** The app exposes no metrics yet.
