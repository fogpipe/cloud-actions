# fogpipe/cloud-actions

GitHub Actions for deploying to **Fogpipe Cloud** from CI using **OIDC
workload-identity federation**: no long-lived secrets stored in your repository.

Your workflow proves its identity with a GitHub OIDC token, exchanges it for a
short-lived, project-scoped credential, and from there either runs the
`fpcloud` CLI directly or uses the actions below.

## Actions

| Action | What it does |
| --- | --- |
| [`auth`](./auth) | Exchange the job's GitHub OIDC token for a short-lived `FPCLOUD_API_KEY`. |
| [`setup`](./setup) | Install a released `fpcloud` on the runner, checksum-verified. After it, any `run: fpcloud …` step works. |
| [`registry-login`](./registry-login) | `docker login` via the registry token broker: a short-lived, IAM-scoped token; push only to your `<org>/<project>/**`. |
| [`config`](./config) | Set the app's config (plain env). A credential is a project secret mounted as a file, set with `fpcloud secret`, not here. |
| [`deploy`](./deploy) | Create or update an app with a new image, named by `release` so it can be rolled back to. If the app has a `release_command`, waits for it and fails the step when it fails. |
| [`website`](./website) | Deploy a directory as a static website: a new version uploaded, then served atomically, earlier versions kept for rollback. |

Pin to the `v1` tag. It moves with every compatible release of these actions;
`@main` is whatever was pushed last.

## Deploy an app

```yaml
name: deploy
on:
  push:
    tags: ['v*']

permissions:
  id-token: write   # required: lets the job mint a GitHub OIDC token
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7

      - uses: fogpipe/cloud-actions/auth@v1
        with:
          service-account: deployer@myproject.k3f.cloud.fogpipe.com

      - id: registry
        uses: fogpipe/cloud-actions/registry-login@v1

      - name: Image ref
        run: echo "IMAGE=${{ steps.registry.outputs.repository }}/myapp:${{ github.ref_name }}" >> "$GITHUB_ENV"

      - uses: docker/setup-buildx-action@v4

      - name: Build & push
        uses: docker/build-push-action@v7
        with:
          context: .
          push: true
          tags: ${{ env.IMAGE }}

      # optional: set config before deploy so new pods boot with it
      - uses: fogpipe/cloud-actions/config@v1
        with:
          project: myproject
          app: myapp
          env: |
            LOG_LEVEL=info

      - uses: fogpipe/cloud-actions/deploy@v1
        with:
          project: myproject
          app: myapp
          image: ${{ env.IMAGE }}
          release: ${{ github.ref_name }}   # what `fpcloud app rollback` aims at
          port: "8080"
          ingress: all
```

`release` is the version the image carries, here the tag that triggered the
run. Leave it out and the revision has no name: it still deploys, but
`fpcloud app rollback <release>` has nothing to aim at.

## Deploy a static website

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - run: npm ci && npm run build

      - uses: fogpipe/cloud-actions/auth@v1
        with:
          service-account: deployer@myproject.k3f.cloud.fogpipe.com
      - uses: fogpipe/cloud-actions/setup@v1

      - uses: fogpipe/cloud-actions/website@v1
        with:
          project: myproject
          name: docs
          dir: dist
```

The first run creates the website; every run uploads the directory as a new
version and switches serving to it. `fpcloud website rollback docs` goes back.

## Use the CLI directly

`setup` puts `fpcloud` on the path and `auth` gives it a credential, so
anything the CLI does is one `run:` step away:

```yaml
      - uses: fogpipe/cloud-actions/auth@v1
        with:
          service-account: deployer@myproject.k3f.cloud.fogpipe.com
      - uses: fogpipe/cloud-actions/setup@v1
        with:
          version: v0.207.4        # optional; default is the latest release
      - run: fpcloud --project myproject app logs myapp --since 1h
```

## One-time setup

Create a service account, grant it the two roles deploying needs on your
project, and trust your repo:

```bash
fpcloud --project myproject sa create deployer

fpcloud --project myproject iam set \
  --member serviceAccount:deployer@myproject.k3f.cloud.fogpipe.com --role app.deployer
fpcloud --project myproject iam set \
  --member serviceAccount:deployer@myproject.k3f.cloud.fogpipe.com --role registry.writer

fpcloud --project myproject federation github add \
  --repo myorg/myrepo --ref refs/tags/* \
  --service-account deployer@myproject.k3f.cloud.fogpipe.com
```

A service account's address is `<name>@<project>.<org short id>.cloud.fogpipe.com`;
`fpcloud sa create` prints it. Nothing is copied into GitHub. `--ref` pins which refs may deploy; omit it to
allow any ref of the repository, including pull requests. A job that deploys
through a GitHub environment presents the environment instead of the ref, so
trust it with `--subject repo:myorg/myrepo:environment:production`. Revoke a
repo at any time with `fpcloud federation remove <id>`.

## Security

Each repo can only act within its own project and can only push to its own
`<org>/<project>/**` registry prefix: a leaked credential cannot touch the
platform's images or another tenant's repositories. Credentials are minted per
run and expire on the trust binding's TTL (15 minutes by default); there is no
stored key to rotate or leak.

## License

[Apache-2.0](./LICENSE), the licence every public Fogpipe repository carries.
