<!-- GENERATED from src/actions/README.md by build/generate.py — edit the source, not this. -->
# Supervisory gates for GitHub Actions

Composite actions that submit software-delivery evidence to the Supervisory service and fail the job when a
control the service **enforces** does not pass. The service owns enforcement: these actions report and obey,
they do not decide.

Pin a released tag, never `@main` — an action is a contract. The versions below are the release this copy
of the documentation was published with, so an example can be copied as it stands.

## The gates

| Action | Controls | When it runs |
|---|---|---|
| `supervise-pr-gate` | SD-2.4.1 static analysis · SD-2.4.2 tests · SD-2.4.3 security scan · SCM-3.1 dependency sources | on the pull request |
| `supervise-build-gate` | SD-1.1 … SD-1.5 | after the container image is pushed |
| `supervise-build-gate-artifact` | SD-1.1 … SD-1.5 | after the artifact is published to S3 or CodeArtifact |
| `supervise-deploy-gate` `phase: pre` | SD-3.4 deploy window · SD-2.5 automated deployment | before deploying |
| `supervise-deploy-gate` `phase: post` | SD-1.7 artifact source · SD-1.8 environment-agnostic artifact | after deploying |

Together these are every control a pipeline submits. The SCM controls (SCM-1.2.x, SCM-1.4, SCM-1.5) are not
here because no pipeline submits them: they are read from GitHub by the supervisory audit collector, which
needs nothing from your workflow.

## What every job needs

```yaml
permissions:
  id-token: write # REQUIRED — mints the OIDC token the evaluation API authenticates
  contents: read
```

`id-token: write` goes on the **calling workflow or job**; a composite action cannot request a permission
for itself. Without it there is no token endpoint in the environment and the gate fails closed, saying so.

Your repository must also be registered with the supervisory control plane, with an identity that accepts
tokens from this GitHub host. Ask the delivery-control owners; nothing in the workflow can substitute for it.

## Pull-request gate

```yaml
name: Supervisory PR gate
on: pull_request

jobs:
  supervise:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
      pull-requests: write # optional — lets an advisory failure leave a note on the PR
    steps:
      - uses: actions/checkout@v4

      # The reports the gate assesses. Earlier jobs upload them; this downloads them back to the same
      # paths. Take ONLY this component's artifacts — in a monorepo, pulling everything fills the runner's
      # disk with reports the gate does not assess.
      - uses: actions/download-artifact@v4
        with: { name: kernel-reports, path: . }

      - uses: syberry-corporation/delivery-control-actions/supervise-pr-gate@1.4.19
        with:
          supervisory_eval_url: ${{ vars.SUPERVISORY_EVAL_URL }}
          supervisory_project: davinci
          component: kernel
```

## Build gate

The container gate reads the image back **out of the registry** — its own SBOM and provenance attestations
— so it assesses what was published rather than what the build claimed. Log in to the registry first.

```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ vars.ECR_ROLE_ARN }}
    aws-region: eu-central-1
- uses: aws-actions/amazon-ecr-login@v2

- uses: syberry-corporation/delivery-control-actions/supervise-build-gate@1.4.19
  with:
    supervisory_eval_url: ${{ vars.SUPERVISORY_EVAL_URL }}
    supervisory_project: davinci
    component: kernel
    image_repository: ${{ vars.ECR_REGISTRY }}/kernel
    image_reference: ${{ github.sha }}
```

For a build that publishes an artifact instead of an image, use `supervise-build-gate-artifact` and give it
the build's metadata file — that file is the build telling the gate what it published and where:

```yaml
- uses: syberry-corporation/delivery-control-actions/supervise-build-gate-artifact@1.4.19
  with:
    source: s3 # or: codeartifact
    supervisory_eval_url: ${{ vars.SUPERVISORY_EVAL_URL }}
    supervisory_project: davinci
    component: frontend
    metadata_file: build-metadata.env
```

## Deploy gate

Two phases, and the split is the point. **Pre** runs before anything is deployed, so an out-of-window or
manual deploy is stopped while stopping it is still free. **Post** runs after, over what the deploy reports
it actually published.

```yaml
deploy:
  runs-on: ubuntu-latest
  environment: prod # puts `environment` in the OIDC token — this is the DEPLOYMENT subject
  permissions:
    contents: read
    id-token: write
  steps:
    - uses: syberry-corporation/delivery-control-actions/supervise-deploy-gate@1.4.19
      with:
        phase: pre
        supervisory_eval_url: ${{ vars.SUPERVISORY_EVAL_URL }}
        supervisory_project: davinci
        component: kernel
        environment: prod

    - run: ./deploy.sh > response.json # must report what was actually published

    - uses: syberry-corporation/delivery-control-actions/supervise-deploy-gate@1.4.19
      with:
        phase: post
        supervisory_eval_url: ${{ vars.SUPERVISORY_EVAL_URL }}
        supervisory_project: davinci
        component: kernel
        environment: prod
        deploy_response: response.json
```

### What your deploy must report

The **pre** phase needs nothing from you: SD-3.4 and SD-2.5 read only what the token proves, plus the
`hotfix` inputs.

The **post** phase is evaluated over a JSON file your deploy writes — `deploy_response`. Our own deploy
Lambdas produce it; if you deploy some other way (helm, terraform, a script, anything), you write it. The
shape is a contract, and this is all of it.

**A container deployment** — `archetype: container`:

```json
{
  "image": {
    "name": "<ECR_REGISTRY>/kernel",
    "digest": "sha256:3f79bb7b435b05321651daefd374cdc681dc06faa65e374e38337b88ca046dea",
    "version": "1.4.2"
  },
  "runtimeConfig": { "envVarNames": ["DB_URL", "LOG_LEVEL"] },
  "deployment": { "id": "…", "automated": true }
}
```

**An artifact deployment** — `archetype: artifact`:

```json
{
  "artifact": {
    "digest": "sha256:3f79bb7b435b05321651daefd374cdc681dc06faa65e374e38337b88ca046dea",
    "location": "s3://acme-prod-s3-artifacts/frontend/1.4.2.zip",
    "version": "1.4.2"
  },
  "runtimeConfig": { "envVarNames": ["API_BASE_URL"] },
  "deployment": { "id": "…" }
}
```

| Field | Read by | Must be |
|---|---|---|
| `image.digest` / `artifact.digest` | SD-1.7 | `sha256:` + 64 lowercase hex |
| `image.name` | SD-1.7 | starts with an approved repository prefix (project configuration) |
| `artifact.location` | SD-1.7 | an `s3://` URL whose bucket carries the trusted artifact-bucket suffix |
| `image.version` / `artifact.version` | SD-1.8 | matches the deploy version pattern — `1.4.2`, `v1.4.2`; **no environment in it** |
| `runtimeConfig.envVarNames` | SD-1.8 | the NAMES of the configuration supplied from outside. Values are not wanted and must not be sent. |
| `deployment` | nobody | carried into the audit record; no control reads it. Put what helps you read a decision later. |

`artifact.inventory` is the one field you do not write: the action adds it from `artifact_metadata_file`,
because listing what is inside the artifact means unpacking it and only the build has done that.

**The digest must be yours.** Compute it from what you deployed — the bytes you uploaded, the image the
platform reports as running. Copying the build's digest passes the schema and asserts nothing: the build's
digest describes what the build uploaded, and pinning by digest exists precisely so nobody has to trust what
happened between that upload and this deploy. A deployer restating the builder is not a second witness, and
the policy cannot tell the difference, because all it can check is that a digest is well-formed.

**When it does not hold**

| Reason code | What it means |
|---|---|
| `DEPLOYED_ARTIFACT_NOT_PINNED_BY_DIGEST` | no digest, or not `sha256:`+64 hex |
| `DEPLOYED_ARTIFACT_NOT_FROM_APPROVED_REPOSITORY` | the image name / S3 bucket is not an approved one |
| `DEPLOYED_ARTIFACT_LOCATION_MISSING` | an artifact deployment with no `location` |
| `DEPLOYED_ARTIFACT_VERSION_MISSING` / `…_NOT_ENV_AGNOSTIC` | no version, or one that names an environment |
| `RUNTIME_CONFIG_NOT_EXTERNALIZED` | `envVarNames` is empty, and nothing proves there is no configuration to externalize |
| `RUNTIME_CONFIG_SHIPPED_IN_ARTIFACT` | the artifact contains a config file and the deploy supplies none from outside |

A deployment that reports nothing fails these closed. That is the control working: before 1.2.0 the gate
filled the gaps from the build's own metadata, which passed SD-1.7 and SD-1.8 while establishing nothing.

Declare `environment:` on the job. It is what puts the `environment` claim in the OIDC token, and the
service binds the deployment subject to that claim rather than to anything the payload says — so a deploy
cannot claim a laxer environment's window.

## Reading a failure

```
── SD-2.4.2 → HTTP 200 ──
[SD-2.4.2] ADVISORY 'FAIL' — not blocking
```

- **PASS** — the control holds.
- **ADVISORY 'FAIL'** — it does not hold, but the posture is advisory, so the job stays green. It carries a
  date after which it starts blocking; with `pull-requests: write` the gate leaves that on the PR so it is
  not quietly ignored.
- **effect BLOCK** — enforced and failing. The job fails; fix the finding, not the gate.
- **HTTP 4xx/5xx** — the gate fails closed. A control that could not be evaluated is not a control that
  passed.

Two failures point at the workflow rather than at your code: *"missing `permissions: id-token: write`"* and
*"Evidence repository does not match authenticated identity"* — the latter means the repository is not the
one this supervisory project is registered for.

## Differences from the GitLab components

The same shell runs on both forges; what differs is what the platform arranges for you.

- **Reports are not inherited.** GitLab's gate receives earlier jobs' artifacts automatically; here the
  workflow downloads them.
- **`id-token: write` is explicit**, per workflow or job.
- **A pull-request run names the head commit.** `GITHUB_SHA` on a `pull_request` event is a merge commit
  that exists in no branch; the evidence carries the head sha, so build traceability points at a commit that
  can be checked out.

---

This repository is published automatically at every release. Changes made here directly are overwritten
by the next one.

## License

[Apache-2.0](LICENSE).
