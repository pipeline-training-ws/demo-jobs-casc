NOTE: I havent tested to apply the jobs with CASC. But I have used the same configuration to setup the demo in UI. (I just exported the config after setup the demo manually)

TODO: For the next workshop a demo controller should be provionied by casc, the casdc resources might require some litle changes/adjustments

# CasC Bundle — Workshop Jobs

This is the Configuration-as-Code (CasC) bundle used to provision the demo/lab jobs for the
CloudBees CI Pipeline Training workshop. See [`items.yaml`](items.yaml) for the full job
definitions, [`jenkins.yaml`](jenkins.yaml) for global configuration (credentials, Template
Catalog registration, build discarders), and [`rbac.yaml`](rbac.yaml) for roles/permissions.

## Bundle Contents

| File | Purpose |
| --- | --- |
| [`bundle.yaml`](bundle.yaml) | Bundle descriptor — wires the sections below together |
| [`jenkins.yaml`](jenkins.yaml) | Global CasC config: credentials, Template Catalog registration, build discarders, GitHub config |
| [`plugins.yaml`](plugins.yaml) | Required plugin list |
| [`rbac.yaml`](rbac.yaml) | Roles (`administer`, `develop`, `browse`) and folder-level role bindings |
| [`items.yaml`](items.yaml) | The job/folder items provisioned on the controller (see below) |

## Prerequisites

Before applying this bundle (or reloading it via `<CONTROLLER_URL>/casc-bundle-mgnt/reload`),
make sure the following are in place:

| Requirement | Used by |
| --- | --- |
| Env var `GITHUB_APP_ID` / `GITHUB_APP_PRIVATE_KEY` | `gh-app` credential (GitHub App, `jenkins.yaml`) |
| Env var `GITHUB_PAT` / `GITHUB_USERNAME` | `gh-pat` credential (username/password, `jenkins.yaml`) |
| Env var `JENKINS_CONTROLLER_URL` | GitHub webhook URL + `unclassified.location.url` |
| GitHub App installed on the `pipeline-training-ws` org with access to `sample-app-helloWorld`, `template-catalog`, `shared-library` | `gh-app`-based jobs (Multibranch, Org Folder, Template Catalog) |
| A `dev1` user known to the configured security realm (`operationsCenter`) | `RBAC-Folder-dev1-only` |
| A Kubernetes cloud with nodes labeled/tolerated for `workload: agent` | `06-parallel-samples`, `07-crossteam-collaboration-events`, `08-checkpoints` |

## Job Setup by Workshop Segment

The jobs in `items.yaml` map onto the hands-on lab segments in [`../labs/`](../labs/). Set them up
in this order — each segment's jobs build on concepts from the previous one.

### Segment 1 — CloudBees CI Architecture & CasC Foundation

No external repo/credentials needed — these jobs validate that the CasC bundle itself loaded
correctly and introduce the Kubernetes agent model.

1. **`00-HeloWorld`** — inline `pipeline { agent none }` script, no SCM.
   - Apply/reload the bundle, then open the job and click **Build Now**.
   - Confirm the console output prints `Hello World`. This is the fastest smoke test that
     `items.yaml` was parsed and synced onto the controller.
2. **`06-parallel-samples` folder → `agent-global`** — single Kubernetes pod (`maven` + `node`
   containers) shared by both parallel branches.
   - Confirm the cluster has a node pool with the `workload: agent` label/toleration used in the
     pod YAML.
   - Run the job and inspect the pod in the Kubernetes tab: one pod, two sibling containers.
3. **`06-parallel-samples` folder → `agent-per-stage`** — `agent none` at the top level; each
   parallel branch requests its own pod.
   - Run the job and confirm **two separate pods** are provisioned (one per branch), each sized
     independently — contrast with `agent-global`'s single shared pod.
4. **`RBAC-Folder-dev1-only`** — folder scoped to the `develop`/`browse` roles from `rbac.yaml`,
   granted to a group containing user `dev1`.
   - Confirm `dev1` exists in the security realm before the bundle reload (RBAC bindings for an
     unknown user are silently ignored).
   - Log in as `dev1` and verify the folder is visible with **Develop** permissions, while other
     folders in the bundle are not.

### Segment 2 — Shared Library Implementation

1. **`01-pipeline-simple`** — Pipeline-from-SCM job (`cpsScmFlowDefinition`) pointing at
   [`pipeline-training-ws/sample-app-helloworld`](https://github.com/pipeline-training-ws/sample-app-helloworld)'s `Jenkinsfile`.
   - Requires the `gh-pat` credential (already provisioned by `jenkins.yaml`) to check out the repo.
   - That `Jenkinsfile` dynamically loads the `shared-library` repo (`SHAREDLIB_GIT_*` env vars)
     and calls the `pipelineTemplateHelloWorld` step — see
     [`../samples/sample-app-helloWorld/README.md`](../samples/sample-app-helloWorld/README.md)
     for the full call chain.
   - Run the job and confirm the `Load Config`, `Hello World`, and (on `main`) `Hi` stages
     execute, printing values sourced from `ci-config.yaml`.

### Segment 3 — Pipeline Types and Templates

All jobs in this segment read from the `pipeline-training-ws` org via the `gh-app` credential —
confirm the GitHub App installation covers `sample-app-helloWorld` and `template-catalog` first.

1. **`02-MultiBranchJob-HelloWorld`** — standard Multibranch Pipeline scanning
   `sample-app-helloWorld`, using the repo's own `Jenkinsfile` (`workflowBranchProjectFactory`).
   - Save the job, then **Scan Repository Now**.
   - Confirm a child job is created for `main` (and any open PRs/forks per the discovery traits).
2. **`02-MultiBranchJob-HelloWorld-customMarker`** — Multibranch job on the same repo, but using
   `customBranchProjectFactory` with `marker: ci-config.yaml`: only branches containing that file
   get a child job, and the pipeline definition itself is loaded from
   `template-catalog`'s `templates/1-helloWorld-MB/Jenkinsfile` (needs the `gh-pat` credential to
   check out `template-catalog`).
   - Scan the job and confirm child jobs only appear for branches that carry `ci-config.yaml`.
   - Remove/re-add `ci-config.yaml` on a test branch and re-scan to see the child job
     disappear/reappear.
3. **`03-GitHub-Organisation`** — Organization Folder scanning the entire `pipeline-training-ws`
   org, with two marker-based project factories:
   - `ci-config.yaml` marker → runs `template-catalog`'s `templates/1-helloWorld-MB/Jenkinsfile`
     (same pattern as job 2 above, applied org-wide).
   - `ci-config-1.yaml` marker → runs a minimal inline `echo 'Hello World'` pipeline.
   - Save the folder and let the `periodicFolderTrigger` (1d) or a manual **Scan Organization
     Folder Now** run; confirm repos/branches are picked up according to whichever marker file
     they contain.
4. **`05-PTC-job-MB`** — a `cloudbeesTemplatedJob` instantiated from the **Pipeline Template
   Catalog Examples** catalog (registered globally in `jenkins.yaml` →
   `globalCloudBeesPipelineTemplateCatalog`, backed by
   [`../samples/template-catalog/catalog.yaml`](../samples/template-catalog/catalog.yaml)).
   - Confirm the catalog has synced (**Manage Jenkins → Pipeline Template Catalogs**) before this
     item is created — otherwise the `model:` reference in `items.yaml` won't resolve.
   - The item is pre-filled with the `1-helloWorld-MB` template and attributes
     (`githubToken: gh-app`, `repoName: sample-app-helloWorld`, `organisation: pipeline-training-ws`).
     If the catalog's internal template ID drifts, recreate this item from **New Item → Pipeline
     Template Catalog Examples → Template 1 helloWorld-MB** with the same attribute values.
   - Run the job and confirm it behaves like job 2/3 above, but built through the Template
     Catalog UI flow instead of a hand-written `customBranchProjectFactory`.

### Segment 4 — CloudBees Pipeline Features

1. **`08-checkpoints`** — two Kubernetes-agent stages (`Say Hello`, `Say Good By`) separated by
   `checkpoint 'Hello'` and `checkpoint 'ByBy'`.
   - Run the job once to completion.
   - Open the build, use **Restart from Checkpoint → Hello**, and confirm the new build skips
     straight to `Say Good By` instead of re-running `Say Hello`.
2. **`07-crossteam-collaboration-events` folder** — event-driven trigger demo:
   - **`Event-producer`** publishes a `com.example:supply-chain-webapp:*-SNAPSHOT` event via
     `publishEvent`.
   - **`Event-consumer-1`** and **`Event-consumer-2`** both subscribe via
     `eventTrigger jmespathQuery("contains(event, 'com.example:') && contains(event, '-SNAPSHOT')")`.
   - Setup: create/enable all three jobs first (the consumers' triggers must be registered before
     the event fires), then run **`Event-producer`** manually.
   - Confirm both consumer jobs fire automatically off the single published event.

## Applying Changes

After editing any file in this bundle:

```
<CONTROLLER_URL>/casc-bundle-mgnt/reload
```

Then verify the reload succeeded on the bundle management page before running any of the jobs
above — a failed reload leaves the previous bundle state in place, so job/credential changes may
silently not have taken effect.
