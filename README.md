# fogpipe/actions

Public GitHub Actions for deploying to **Fogpipe Cloud** from CI using **OIDC
workload-identity federation** — no long-lived secrets stored in your repository.

| Action | Purpose |
| --- | --- |
| `fogpipe/actions/cloud-auth` | Exchange the job's GitHub OIDC token for a short-lived `FPCLOUD_API_KEY`. |
| `fogpipe/actions/registry-login` | Docker-login to the Fogpipe registry with a **project-scoped** credential (push only to `tenants/<project>/**`). |
| `fogpipe/actions/deploy` | Create or update a Fogpipe app with a new image. |

```yaml
permissions: { id-token: write, contents: read }
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: fogpipe/actions/cloud-auth@main
        with: { service-account: deployer@myproject.cloud.fogpipe.com }
      - id: registry
        uses: fogpipe/actions/registry-login@main
      - run: |
          IMG="${{ steps.registry.outputs.repository }}/app:${{ github.sha }}"
          docker build -t "$IMG" . && docker push "$IMG"
          echo "IMAGE=$IMG" >> "$GITHUB_ENV"
      - uses: fogpipe/actions/deploy@main
        with: { project: myproject, app: app, image: "${{ env.IMAGE }}" }
```
