# Deployment

Safe Settings is deployed to the existing GKE target by
`.github/workflows/deploy-k8s.yml`. A push to `main-enterprise` automatically
deploys application, image, chart, or workflow changes after unit tests pass.
Changes limited to `.github/repos/*.yml` do not rebuild the application.

Deployment is deliberately disabled until the repository variable
`SAFE_SETTINGS_DEPLOY_ENABLED` is set to `true`.

## One-time GitHub Actions setup

Create a `production` environment and make these secrets available to the
repository, either by sharing the existing Hacktron organization secrets or by
adding repository/environment secrets with the same names:

- `GCP_WORKLOAD_IDENTITY_PROVIDER`
- `GCP_SERVICE_ACCOUNT`

The Workload Identity provider must map `attribute.repository` and trust
`HacktronAI/safe-settings`. Use the dedicated service account
`safe-settings-deployer@hacktron-462816.iam.gserviceaccount.com`; do not reuse a
broad infrastructure runner. It needs:

- `roles/artifactregistry.writer` on only the `safe-settings` Artifact Registry
  repository
- `roles/container.clusterViewer` on project `hacktron-462816`
- `roles/iam.workloadIdentityUser` granted to the GitHub OIDC principal on the
  service account

Create the dedicated namespace and apply its scoped Kubernetes role once from
an administrator context:

```bash
kubectl apply --filename helm/safe-settings/namespace.yaml
kubectl apply --filename helm/safe-settings/deployer-rbac.yaml
```

This limits the CI identity to the Helm resources used by this chart in the
`safe-settings` namespace instead of granting project-wide Kubernetes
developer access. Helm stores its release metadata in ConfigMaps so CI does not
need permission to read Kubernetes Secrets.

The workflow currently targets:

- project: `hacktron-462816`
- registry: `us-central1-docker.pkg.dev/hacktron-462816/safe-settings`
- cluster: `safe-settings-cluster`
- cluster location: `us-central1-a`
- Helm release and deployment: `safe-settings`
- namespace: `safe-settings`

Only enable deployment after those resources and permissions are confirmed:

```bash
gh variable set SAFE_SETTINGS_DEPLOY_ENABLED \
  --repo HacktronAI/safe-settings \
  --body true
```

## One-time runtime secret setup

The workflow never copies GitHub App credentials into an image or command
line. The pod reads them from the `safe-settings/app-env` Kubernetes Secret.
The required keys are:

- `APP_ID`
- `PRIVATE_KEY`
- `WEBHOOK_SECRET`
- `WEBHOOK_PROXY_URL` while Smee is used

After authenticating to GCP and selecting the cluster, populate the Secret from
the ignored local `.env` file:

```bash
gcloud auth login
gcloud container clusters get-credentials safe-settings-cluster \
  --project hacktron-462816 \
  --zone us-central1-a
kubectl apply --filename helm/safe-settings/namespace.yaml
K8S_NAMESPACE=safe-settings ./script/bootstrap-k8s-secret
```

The bootstrap script writes values only to a private temporary directory,
applies the Secret, and removes the temporary files without printing values.

## Webhooks

The GitHub App must have an active runtime and a webhook transport. The current
App configuration points to Smee. Keeping `WEBHOOK_PROXY_URL` in `app-env`
causes Probot to connect outbound to that Smee channel; a public Kubernetes
Ingress is not required for this initial setup. The webhook endpoint inside the
application is `/api/github/webhooks`.

For a direct production webhook later:

1. Provide a DNS hostname and HTTPS certificate.
2. Enable the Helm ingress for that hostname.
3. Change the GitHub App webhook URL to
   `https://<hostname>/api/github/webhooks` with SSL verification enabled.
4. Remove `WEBHOOK_PROXY_URL` from `.env` and run
   `DIRECT_WEBHOOK=true ./script/bootstrap-k8s-secret`; the explicit flag
   removes the proxy key from `app-env`.
5. Send a test delivery and confirm a `2xx` response plus application logs for
   that delivery.

Smee returning `200` only confirms that Smee accepted a GitHub delivery; it
does not prove a Safe Settings pod was connected to consume it.

## First deployment cutover

The legacy release runs as `default/safe-settings`. Do not run both releases
against the same Smee channel because a delivery could be processed twice.
After this workflow is merged, perform the first deployment as a short,
reversible cutover:

1. Keep `SAFE_SETTINGS_DEPLOY_ENABLED=false` while merging the workflow.
2. Scale `default/safe-settings` to zero replicas.
3. Set `SAFE_SETTINGS_DEPLOY_ENABLED=true` and manually run the deployment
   workflow from `main-enterprise`.
4. Verify `safe-settings/safe-settings` is ready and its logs show the Smee
   client receiving a delivery.
5. If verification fails, set the variable back to `false` and scale the
   legacy deployment back to one replica.

After a successful cutover, pushes to `main-enterprise` deploy automatically.

## Manual deployment

The Actions workflow can also be started from **Actions → Deploy
safe-settings → Run workflow**. It uses the same tests, immutable image tag,
runtime Secret check, and atomic Helm deployment as automatic pushes.
