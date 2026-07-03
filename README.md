# fogpipe/actions

GitHub Actions for deploying to **Fogpipe Cloud** from CI using **OIDC
workload-identity federation** — no long-lived secrets stored in your repository.

Your workflow proves its identity with a GitHub OIDC token, exchanges it for a
short-lived, project-scoped credential, pushes its image to its own slice of the
registry, and rolls out the app.

## Actions

| Action | What it does |
| --- | --- |
| [`cloud-auth`](./cloud-auth) | Exchange the job's GitHub OIDC token for a short-lived `FPCLOUD_API_KEY`. |
| [`registry-login`](./registry-login) | `docker login` with a **project-scoped** credential — push only to `tenants/<project>/**`. |
| [`config`](./config) | Set app config (ConfigMap) + secrets (Secret) for the project's app. |
| [`deploy`](./deploy) | Create or update a Fogpipe app with a new image. |

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

      - uses: fogpipe/actions/cloud-auth@main
        with:
          service-account: deployer@myproject.cloud.fogpipe.com

      - id: registry
        uses: fogpipe/actions/registry-login@main

      - name: Build & push
        run: |
          IMG="${{ steps.registry.outputs.repository }}/myapp:${{ github.sha }}"
          docker build -t "$IMG" .
          docker push "$IMG"
          echo "IMAGE=$IMG" >> "$GITHUB_ENV"

      # optional: set config + secrets before deploy so new pods boot with them
      - uses: fogpipe/actions/config@main
        with:
          project: myproject
          app: myapp
          env: |
            LOG_LEVEL=info
          secret: |
            API_TOKEN=${{ secrets.API_TOKEN }}

      - uses: fogpipe/actions/deploy@main
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

Full guide: [deploy-from-github](https://github.com/fogpipe/cloud/blob/main/docs/deploy-from-github.md).
