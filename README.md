# fogpipe/cloud-actions

GitHub Actions for deploying to **Fogpipe Cloud** from CI using **OIDC
workload-identity federation** — no long-lived secrets stored in your repository.

Your workflow proves its identity with a GitHub OIDC token, exchanges it for a
short-lived, project-scoped credential, pushes its image to its own slice of the
registry, and rolls out the app.

## Actions

| Action | What it does |
| --- | --- |
| [`auth`](./auth) | Exchange the job's GitHub OIDC token for a short-lived `FPCLOUD_API_KEY`. |
| [`registry-login`](./registry-login) | `docker login` via the registry token broker — a short-lived, IAM-scoped token; push only to your `<org>/<project>/**`. |
| [`config`](./config) | Set app config (ConfigMap) + secrets (Secret) for the project's app. |
| [`deploy`](./deploy) | Create or update a Fogpipe Cloud app with a new image. |

## Usage

```yaml
name: deploy
on:
  push:
    branches: [main]

permissions:
  id-token: write   # required — lets the job mint a GitHub OIDC token
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7

      - uses: fogpipe/cloud-actions/auth@main
        with:
          service-account: deployer@myproject.cloud.fogpipe.com

      - id: registry
        uses: fogpipe/cloud-actions/registry-login@main

      - name: Image ref
        run: echo "IMAGE=${{ steps.registry.outputs.repository }}/myorg/myapp:${{ github.sha }}" >> "$GITHUB_ENV"

      - uses: docker/setup-buildx-action@v4

      - name: Build & push
        uses: docker/build-push-action@v7
        with:
          context: .
          push: true
          tags: ${{ env.IMAGE }}

      # optional: set config + secrets before deploy so new pods boot with them
      - uses: fogpipe/cloud-actions/config@main
        with:
          project: myproject
          app: myapp
          env: |
            LOG_LEVEL=info
          secret: |
            API_TOKEN=${{ secrets.API_TOKEN }}

      - uses: fogpipe/cloud-actions/deploy@main
        with:
          project: myproject
          app: myapp
          image: ${{ env.IMAGE }}
          port: "8080"
          ingress: all
```

## One-time setup

Create a service account, grant it a role on your project, and trust your repo:

```bash
fpcloud --project myproject sa create deployer

fpcloud --project myproject iam set \
  --member serviceAccount:deployer@myproject.cloud.fogpipe.com --role editor

fpcloud --project myproject federation github add \
  --repo myorg/myrepo \
  --service-account deployer@myproject.cloud.fogpipe.com
```

Nothing is copied into GitHub. Scope tighter with `--ref refs/tags/*` to deploy
only on tags, and revoke a repo instantly with `fpcloud federation remove <id>`.

## Security

Each repo can only act within its own project and can only push to its own
`tenants/<project>/**` prefix — a leaked credential can't touch the platform's
images or another tenant's repositories. Credentials are short-lived and minted
per run; there is no stored key to rotate or leak.
