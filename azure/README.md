<!-- GENERATED from src/azure/README.md by build/generate.py — edit the source, not this. -->
# Supervisory gates for Azure Pipelines

Step templates that submit software-delivery evidence to the Supervisory service and fail the job when a
control the service **enforces** does not pass. The service owns enforcement: these templates report and
obey, they do not decide. They run the same shell as the GitLab components and the GitHub actions in this
repository, so a component scores the same on every forge.

Pin a released tag, never `main` — a template is a contract. The tag below is the release this copy of the
documentation was published with, so an example can be copied as it stands.

## The templates

| Template | Controls | When it runs |
|---|---|---|
| `azure/supervise-pr-gate.yml` | SD-2.4.1 static analysis · SD-2.4.2 tests · SD-2.4.3 security scan · SCM-3.1 dependency sources · SCM-3.3 lock files | on the pull request |
| `azure/supervise-build-gate.yml` | SD-1.1 … SD-1.5 · SCM-2.2.2 peer review | after the container image is pushed |
| `azure/supervise-build-gate-artifact.yml` | SD-1.1 … SD-1.5 · SCM-2.2.2 peer review | after the artifact is published to S3 or CodeArtifact |
| `azure/supervise-deploy-gate.yml` `phase: pre` | SD-3.4 deploy window · SD-2.5 automated deployment | before deploying |
| `azure/supervise-deploy-gate.yml` `phase: post` | SD-1.7 artifact source · SD-1.8 environment-agnostic artifact | after deploying |

Three more produce what the gates read:

| Template | Writes | Read by |
|---|---|---|
| `azure/security-scan.yml` | `supervisory/security-scan.json`, `supervisory/dependency-inventory.json`, `supervisory/dependency-locks.json` | the PR gate |
| `azure/build-image.yml` | the image under a version tag, its SBOM and SLSA provenance as OCI referrers | the container build gate |
| `azure/publish-static-artifact.yml` | `<version>.zip`, its SBOM and provenance in the artifact bucket, and the metadata file | the artifact build gate, the post-deploy gate |

The parameters have the names of the GitHub actions' inputs; the reference for each one is the GitHub
action of the same name ([`../README.md`](../README.md)). The gates add `service_connection`, below.

## .NET

`security-scan.yml` with `toolchain: dotnet` restores and reads every NuGet package with the feed it came
from; what a project has to switch on for SCM-3.3, SCM-3.1, SD-2.4.2 and SD-2.4.1 is in the
[.NET section of the actions' README](../README.md#net). On Azure Pipelines: `UseDotNet@2` when the agent
lacks the SDK you build with, and `NuGetAuthenticate@1` before the scan when restore reads an Azure
Artifacts feed — that feed's prefix is approved per project in its supervisory registration.

## Before the first run

1. **The supervisory project is registered for your service connection.** Azure DevOps mints an OIDC token
   only for a service connection, so that is the identity the evaluation API authenticates. The
   delivery-control team registers it with:
   * the issuer `https://vstoken.dev.azure.com/<organization id>` and the audience
     `api://AzureADTokenExchange` (fixed by Azure; nothing to choose);
   * the subject `sc://<organization>/<project>/<service connection name>`;
   * the environments a deploy may name.

   A token proves which service connection it was minted for, and nothing else — no repository, no
   pipeline, no environment, no commit. Those four are therefore taken from the pipeline run and sent in
   the evidence, and the service accepts an environment only from the registered list.
2. **An Azure Resource Manager service connection with workload identity federation.** The gates run as an
   `AzureCLI@2` step of it, which signs in with it before the script starts — so its identity needs a role
   on the connection's scope (Reader is enough). Grant the pipeline permission to use it.
3. **A Linux agent** — `ubuntu-latest`, or a self-hosted one with bash, curl, jq, the Azure CLI, and docker
   for the image templates. The scripts are bash; nothing here runs on a Windows agent.
4. **A GitHub service connection** in your Azure DevOps project, to read these templates. The repository is
   public, but Azure Pipelines requires an endpoint for any GitHub repository resource; any GitHub
   connection will do.

## Referencing the templates

```yaml
resources:
  repositories:
    - repository: delivery_control
      type: github
      name: syberry-corporation/delivery-control-actions
      ref: refs/tags/1.12.0
      endpoint: <your GitHub service connection>

variables:
  SUPERVISORY_EVAL_URL: https://<evaluation API>
  SUPERVISORY_PROJECT: <supervisory project id>
  SUPERVISORY_CONNECTION: <the registered service connection>
```

A template is then a step: `- template: azure/<name>.yml@delivery_control` with its `parameters:`. The
sections below are the parts of one pipeline — pull request, build, deploy.

## Pull-request gate

```yaml
- job: security_scan
  steps:
    - checkout: self
    - template: azure/security-scan.yml@delivery_control
      parameters:
        toolchain: php            # php | node | java | python
        dockerfile: Dockerfile    # scan_target: filesystem for a component with no image
    - publish: supervisory
      artifact: backend-supervisory
      condition: always()

- job: supervise
  dependsOn: [security_scan]     # and your lint/test jobs, which publish build/ as backend-checks
  condition: always()
  steps:
    - checkout: self
    - task: DownloadPipelineArtifact@2
      inputs:
        artifact: backend-supervisory
        path: $(System.DefaultWorkingDirectory)/supervisory
    - template: azure/supervise-pr-gate.yml@delivery_control
      parameters:
        service_connection: ${{ variables.SUPERVISORY_CONNECTION }}
        supervisory_eval_url: $(SUPERVISORY_EVAL_URL)
        supervisory_project: $(SUPERVISORY_PROJECT)
        component: backend
        test_tool: phpunit
        test_report_format: junit
```

The reports go back to the paths they were written at: the gate reads `supervisory/` and `build/` relative
to `project_path`, as on the other forges. Download only this component's artifacts.

`service_connection` must be known when the pipeline is compiled — a literal, a template parameter, or
`${{ variables.X }}` of a variable defined in the YAML, not one set at queue time — because Azure authorizes
the connection and runs its checks before the job starts.

## Build gate

```yaml
- job: build
  steps:
    - checkout: self
    # log in to the registry first (Docker@2 login, or aws ecr get-login-password | docker login)
    - template: azure/build-image.yml@delivery_control
      parameters:
        image_repository: $(ECR_REGISTRY)/backend
        version: 1.0.$(Build.BuildId)      # environment-agnostic: never a branch or an env name
        dockerfile: Dockerfile
    - template: azure/supervise-build-gate.yml@delivery_control
      parameters:
        service_connection: ${{ variables.SUPERVISORY_CONNECTION }}
        supervisory_eval_url: $(SUPERVISORY_EVAL_URL)
        supervisory_project: $(SUPERVISORY_PROJECT)
        component: backend
        image_repository: $(ECR_REGISTRY)/backend
        image_reference: 1.0.$(Build.BuildId)
```

`build-image.yml` sets two output variables on its step (named `build_image` unless you pass `name:`):
`digest` and `image` (repository@digest) — `$(build_image.digest)` later in the job,
`dependencies.build.outputs['build_image.digest']` in a later job. Deploy by the digest.

For a build that publishes an artifact instead, `publish-static-artifact.yml` (outputs `metadata_file`,
`location`, `digest` on the step `publish_static_artifact`) and then `supervise-build-gate-artifact.yml` with
`source: s3` and `metadata_file: $(publish_static_artifact.metadata_file)`. A build that publishes **no
artifact at all** still runs the artifact gate, with `metadata_file` left out: it submits exactly that, and
the build controls fail on it — the record shows the build and why it does not hold.

## Deploy gate

```yaml
- job: deploy_dev
  steps:
    - template: azure/supervise-deploy-gate.yml@delivery_control
      parameters:
        phase: pre
        service_connection: ${{ variables.SUPERVISORY_CONNECTION }}
        supervisory_eval_url: $(SUPERVISORY_EVAL_URL)
        supervisory_project: $(SUPERVISORY_PROJECT)
        component: backend
        environment: dev

    - bash: ./deploy.sh > response.json   # must report what was actually deployed

    - template: azure/supervise-deploy-gate.yml@delivery_control
      parameters:
        phase: post
        service_connection: ${{ variables.SUPERVISORY_CONNECTION }}
        supervisory_eval_url: $(SUPERVISORY_EVAL_URL)
        supervisory_project: $(SUPERVISORY_PROJECT)
        component: backend
        environment: dev
        deploy_response: response.json
```

The steps work the same inside a `deployment` job (`strategy: runOnce: deploy: steps:`), where an Azure
DevOps environment's approvals and checks run before them.

`environment` is the supervisory environment — one of those registered for the project — not necessarily
the name of an Azure DevOps environment. An unregistered one is refused (403) before any control is
evaluated. The shape of `response.json` is the contract described in
[What your deploy must report](../README.md#what-your-deploy-must-report).

## The token

Each gate asks Azure DevOps for a fresh token right before it submits — `System.OidcRequestUri`, for the
service connection, with the job's `System.AccessToken` — because one lives five minutes and a gate can take
longer; a token that lapses mid-gate (HTTP 401) is minted again once. The connection's id comes from the
`AzureCLI@2` task; set `service_connection_id` only if your agent's task version does not export it. When
no token can be minted, the gate falls back to the one the task received at start, and with neither it
fails closed and says so.

## Securing the service connection

Whoever can run a pipeline with the registered service connection can submit evidence for the supervisory
project, and names the repository, environment and commit it is about. Treat the connection accordingly:

* use a **dedicated** connection for the gates rather than the one your deploys use, so that permission to
  deploy and permission to vouch for a deploy are granted separately;
* open its **pipeline permissions** only to this project's pipelines;
* add a **Branch control** check limited to your protected branches, so a pipeline run from an arbitrary
  branch cannot use it.

## Differences from the GitHub actions

* **The gates take `service_connection`**, and have no `supervisory_audience`: Azure fixes the audience.
* **Outputs are step output variables**, addressed by the step's `name:`.
* **The environment, repository and commit travel in the evidence**, limited server-side as above; on
  GitHub and GitLab the token carries them.
* **So do how the run started and which commit defined it** — `Build.Reason` and `Build.SourceVersion`,
  in every submission's `source`. The service hands them to the policy as the claims a token would carry;
  a `Manual` run is not automated. Nothing verifies them: the decision lists them in
  `authentication.assertedClaims`, and one whose control read them carries the advisory
  `PIPELINE_FACTS_SELF_ASSERTED`, so the record says they are the run's own word.
* **A pull-request build names the source commit and branch**, not the merge commit Azure DevOps checks
  out — so build traceability points at a commit that exists.
* **Advisory notes** are left on Azure Repos pull requests when the build service has "Contribute to pull
  requests" on the repository; a GitHub repository built by Azure Pipelines gets none.
* **Report links point at the build's page**, which lists every job's artifacts.

Reading a decision — PASS, ADVISORY 'FAIL', effect BLOCK, a fail-closed HTTP error — is the same as on GitHub:
see [Reading a failure](../README.md#reading-a-failure).
