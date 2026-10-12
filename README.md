# matelot-cloud/helm-charts

The Helm chart that installs a [Matelot](https://matelot-cloud.github.io/website/) cell — the self-hosted product-security
platform — plus the operator scripts for backups and restores.

| Path | What |
| --- | --- |
| `charts/secops-cell/` | the chart: API, worker, web app, managed runners, built-in registry, Postgres, Redis, Keycloak, backups — one release per organization |
| `scripts/cell-backup.sh`, `scripts/cell-restore.sh` | run a backup now and download it; restore a cell (secrets, databases) from an archive |

The chart is **published as an OCI package**, which is how you install it; this repository is its readable copy. Both
are written by Matelot's release pipeline — the folder here is overwritten on every release (tags `secops-cell-X.Y.Z`),
so pull requests against the chart cannot be merged; report problems in
[matelot-cloud/matelot](https://github.com/matelot-cloud/matelot/issues).

## Install

```bash
helm show values oci://ghcr.io/matelot-cloud/charts/secops-cell --version 0.1.0 > values.yaml
# edit: tenant (slug, name, domain, adminEmails), ingress (className, tls, annotations), postgres.password,
#       keycloak.adminPassword, backup
helm install acme oci://ghcr.io/matelot-cloud/charts/secops-cell --version 0.1.0 \
  --namespace tenant-acme --create-namespace -f values.yaml
kubectl -n tenant-acme get pods -w
```

The full guide — requirements, every value that matters, first administrator, CLI, backups, upgrades, removal — is
the **[install guide](https://matelot-cloud.github.io/website/docs/latest/install.html)**. In short:

### Values that matter

```yaml
tenant:
  slug: acme                      # DNS label: hostnames acme.<domain>, auth-acme.<domain>, registry-acme.<domain>
  name: "ACME Corporation"
  domain: platform.example.com
  adminEmails: ["ciso@acme.com"]  # first login becomes administrator of the root organization
ingress:
  className: nginx
  tls: true
  annotations: { cert-manager.io/cluster-issuer: letsencrypt }
postgres:
  bundled: true                   # or bundled: false + externalUrl for a managed database
  password: "<generate>"
keycloak:
  adminPassword: "<generate>"
backup:
  enabled: true
  schedule: "17 2 * * *"
  pvc: { enabled: true, size: 20Gi }
  s3: { enabled: false }          # optional off-site copy: bucket, region, credentialsSecret or workload identity
```

Images default to `ghcr.io/matelot-cloud/{api,worker,web,runner}` at the chart's `appVersion`.

### Ingress and TLS

Three hostnames per cell: the app and API (`<slug>.<domain>`), Keycloak (`auth-<slug>.<domain>`) and the built-in
registry (`registry-<slug>.<domain>`). `ingress.tls: true` with a cert-manager annotation issues the certificates
into the Secrets `<release>-app-tls`, `<release>-keycloak-tls` and `<release>-registry-tls`; without cert-manager, create
those Secrets yourself with the same names. `ingress.controllerNamespace` is used by the NetworkPolicy. A wildcard DNS
record `*.<domain>` is simplest.

### Database

`postgres.bundled: true` runs Postgres in the cell on a volume. For a managed database set `postgres.bundled: false`,
`postgres.externalUrl` (`postgresql://…/platform`) and `postgres.externalKeycloakJdbcUrl` (`jdbc:postgresql://…/keycloak`);
create both databases first, the schemas come from the migrations. Redis and Keycloak can be external the same way
(`redis.externalUrl`, `keycloak.externalUrl`).

### Backups

With `backup.enabled` a CronJob dumps both databases and the cell's Secrets into one archive on the backups volume
and, optionally, into your S3 bucket. On a StorageClass that binds on first use the backups PVC stays Pending until
the first backup runs — do not use `helm install --wait`; run the first backup when the pods are up:

```bash
kubectl -n tenant-acme create job acme-backup-first --from=cronjob/acme-backup
scripts/cell-backup.sh -r acme -n tenant-acme -d ./backups          # run now + download + checksum
scripts/cell-restore.sh -r acme -n tenant-acme ./backups/acme-….tar.gz
```

The archive contains the encryption key and passwords: restrict access to it. Rebuilding a cell on a new cluster
restores the Secrets **before** `helm install` (`--secrets-only`, install, `--databases-only`) — see the
[backup and restore guide](https://matelot-cloud.github.io/website/docs/latest/backup-restore.html).

### Upgrades

```bash
helm upgrade acme oci://ghcr.io/matelot-cloud/charts/secops-cell --version 0.2.0 --namespace tenant-acme -f values.yaml
```

Back up first. Migrations run in the API's init container; a release rolls only the api, worker, web and runner
pods. Release notes and digests: https://github.com/matelot-cloud/matelot/releases.

## Verifying the chart

```bash
helm pull oci://ghcr.io/matelot-cloud/charts/secops-cell --version 0.1.0
cosign verify ghcr.io/matelot-cloud/charts/secops-cell:0.1.0 \
  --certificate-identity-regexp 'https://github.com/matelot-cloud/platform/' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

## Licence

The chart and scripts are part of Matelot and covered by the [customer licence](LICENSE).
