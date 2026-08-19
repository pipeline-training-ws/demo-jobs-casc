# CloudBees CI Controller Setup Guide

This guide walks through configuring a CloudBees CI **managed controller** with the plugins,
global configuration, credentials, and jobs needed to run the `pipeline-training-2026` workshop
labs. It is derived directly from:

- **[`demo-jobs-casc/`](demo-jobs-casc/)** — the Configuration-as-Code (CasC) bundle exported from
  an already-working controller (`bundle.yaml`, `jenkins.yaml`, `plugins.yaml`, `rbac.yaml`,
  `items.yaml`).
- **[`samples/`](samples/)** — the four GitHub repositories the jobs actually build against
  (`sample-app-helloWorld`, `template-catalog`, `shared-library`, `pipeline-samples`).

---

## 1. Architecture Overview

```
GitHub org: pipeline-training-ws
├── sample-app-helloWorld   (app repo, ci-config.yaml + Jenkinsfile)
├── template-catalog        (Pipeline Template Catalog: 0-helloWorld, 1-helloWorld-MB)
├── shared-library           (Jenkins Shared Library: pipelineTemplateHelloWorld, etc.)
└── pipeline-samples         (reference Jenkinsfiles: checkpoints, parallel, cross-team, jfrog, jira)

CloudBees CI
├── Operations Center (CJOC)         — securityRealm: operationsCenter (see §7)
│   └── Managed Controller           — this is what demo-jobs-casc/ configures
│       ├── CasC bundle (bundle.yaml → jenkins.yaml + plugins.yaml + rbac.yaml + items.yaml)
│       ├── Kubernetes cloud          — NOT included in the bundle, see §5 (external prerequisite)
│       └── Jobs (items.yaml)         — see §8
```

`jenkins.yaml` sets `securityRealm: "operationsCenter"`, i.e. this controller **delegates**
authentication to its parent Operations Center rather than defining its own realm. This guide
therefore assumes an Operations Center already exists and this bundle is applied to a **managed
controller** underneath it — not a standalone Jenkins instance.

---

## 2. Prerequisites

Gather/confirm these before touching the controller:

| # | Requirement | Why it's needed | Source |
| --- | --- | --- | --- |
| 1 | A CloudBees CI Operations Center with a managed controller provisioned | `securityRealm: operationsCenter` requires a parent OC | `demo-jobs-casc/jenkins.yaml:27` |
| 2 | A GitHub App installed on the `pipeline-training-ws` org with access to `sample-app-helloWorld`, `template-catalog`, and `shared-library` | Backs the `gh-app` credential used by Multibranch/Org Folder/Template Catalog jobs | `demo-jobs-casc/jenkins.yaml:6-12`, `items.yaml` |
| 3 | A GitHub Personal Access Token + username with read access to the same repos | Backs the `gh-pat` / `gh-pat-key` credentials used for plain checkout and the classic GitHub webhook plugin | `demo-jobs-casc/jenkins.yaml:13-23` |
| 4 | Values available for `GITHUB_APP_ID`, `GITHUB_APP_PRIVATE_KEY`, `GITHUB_PAT`, `GITHUB_USERNAME`, `JENKINS_CONTROLLER_URL` | These are referenced as `${VAR}` placeholders throughout `jenkins.yaml`; CasC will fail to resolve credentials/URLs without them | `demo-jobs-casc/jenkins.yaml` (multiple lines) |
| 5 | A Kubernetes cloud already configured on the controller, with a node pool labeled/tolerated for `workload: agent` | 6 of the 9 jobs (`01-pipeline-simple` via the shared library, `06-parallel-samples/*`, `07-crossteam-collaboration-events/*`, `08-checkpoints`) run on `agent { kubernetes { ... } }` pods | see §5 |
| 6 | A user named `dev1` known to whatever backing identity source the Operations Center's security realm uses | `RBAC-Folder-dev1-only` grants this user the `develop` role; CasC RBAC bindings for unknown users are silently ignored | `demo-jobs-casc/items.yaml:810-817` |

**Not required for the jobs currently in this bundle** (documented in `samples/` but not wired into
any `items.yaml` job): `jfrog-user-token` and `jira-user-token` credentials, used only by the
standalone `ci-jfrog-integration` / `ci-jira-integration` samples under
`samples/pipeline-samples/`. Skip these unless you plan to add those pipelines as jobs yourself.

---

## 3. Install the Required Plugins

Source: [`demo-jobs-casc/plugins.yaml`](demo-jobs-casc/plugins.yaml). Install via **Manage Jenkins
→ Plugins**, or let the CasC bundle install them automatically on bundle apply (see §4).

| Plugin ID | Purpose in this bundle |
| --- | --- |
| `kubernetes` | Kubernetes pod agents used by most jobs |
| `github-branch-source` | Multibranch/Org Folder GitHub discovery (`02-*`, `03-GitHub-Organisation`) |
| `workflow-aggregator` | Core Pipeline (Declarative + Scripted) support |
| `workflow-cps-checkpoint` | `checkpoint` step (`08-checkpoints`) |
| `pipeline-event-step` | `publishEvent` / `eventTrigger` (`07-crossteam-collaboration-events`) |
| `pipeline-utility-steps` | `readYaml` (used by `pipelineTemplateHelloWorld` in the shared library) |
| `cloudbees-workflow-template` | Pipeline Template Catalog support (`05-PTC-job-MB`, `globalCloudBeesPipelineTemplateCatalog`) |
| `cloudbees-casc-items-controller` | CasC-managed `items.yaml` (jobs/folders as code) |
| `cloudbees-casc-client` | CasC bundle client on the managed controller |
| `cloudbees-build-strategies-plugin` | `skipInitialBuildOnFirstIndexingResetRevision` build strategy used on all branch sources |
| `cloudbees-github-reporting` | `cloudBeesSCMReporting` (GitHub Checks reporting) trait on branch sources |
| `operations-center-cloud` | Managed controller / Operations Center integration |
| `operations-center-notification` | OC notification support |
| `cloudbees-monitoring`, `cloudbees-jenkins-advisor`, `cloudbees-inactive-items`, `cloudbees-nodes-plus`, `cloudbees-groovy-view`, `cloudbees-pipeline-explorer`, `cloudbees-view-creation-filter` | CloudBees platform features enabled globally (see `unclassified.cloudbeesPipelineExplorer` in `jenkins.yaml`) |
| `cloudbees-hashicorp-vault`, `cloudbees-google-cloud-storage-cache`, `cloudbees-s3-cache`, `cloudbees-ssh-slaves` | Available CloudBees integrations — not exercised by any job in `items.yaml`, but listed as installed |
| `config-file-provider` | Config file management (no job in `items.yaml` currently references a managed config file) |
| `email-ext` | Extended email notifications (not referenced by any job in `items.yaml`) |
| `infradna-backup` | Backup plugin |

> `plugins.yaml` itself carries this caveat (line 1): plugins that were **manually uploaded**
> rather than installed through the Plugin Manager or an Update Center may not be installable
> purely from this exported list. If a plugin fails to resolve, check whether it needs to be
> uploaded as a `.hpi`/`.jpi` file instead.

---

## 4. Apply the CasC Bundle

The bundle descriptor wires the four config files together:

```yaml
# demo-jobs-casc/bundle.yaml
apiVersion: "2"
id: "workshop"
version: "1"
jcasc:  ["jenkins.yaml"]
plugins: ["plugins.yaml"]
rbac:   ["rbac.yaml"]
items:  ["items.yaml"]
```

1. Make the five environment variables from §2.4 available to the controller (e.g. as Kubernetes
   Secret-backed env vars on the managed controller pod, or via your CasC secrets mechanism) —
   **before** the bundle loads, since `jenkins.yaml` interpolates them at parse time.
2. In Operations Center, assign this bundle (`demo-jobs-casc/`) to the target managed controller,
   or place its contents at the Git location your controller is configured to pull CasC bundles
   from.
3. Trigger a reload:

   ```
   <CONTROLLER_URL>/casc-bundle-mgnt/reload
   ```

4. **Verify the reload succeeded** on the bundle management page before doing anything else — a
   failed reload silently leaves the previous state in place, so credential/job changes may not
   actually have taken effect.

---

## 5. Global Configuration (`jenkins.yaml`)

### 5.1 Core Jenkins settings

| Setting | Value | Effect |
| --- | --- | --- |
| `authorizationStrategy` | `cloudBeesRoleBasedAccessControl` | RBAC roles from `rbac.yaml` are enforced (see §6) |
| `numExecutors` | `0` | The controller itself provides **no build executors** — every job must resolve an agent from a cloud (Kubernetes) or explicitly declare `agent none` |
| `securityRealm` | `operationsCenter` | Delegates authentication to the parent Operations Center (see §1) |
| `systemMessage` | `Controller configured using CloudBees CasC v1` | Cosmetic |

Because `numExecutors: 0`, confirm the Kubernetes cloud (below) is working **before** running any
job that doesn't explicitly set `agent none` — otherwise builds will queue indefinitely with no
available executor.

### 5.2 Kubernetes cloud — external prerequisite, not in this bundle

`jenkins.yaml` contains **no `clouds:` block**. All pod specs in `items.yaml` use:

```yaml
nodeSelector:
  workload: agent
tolerations:
- key: workload
  operator: Equal
  value: agent
  effect: NoSchedule
```

This means a Kubernetes cloud must already be configured on the controller (commonly inherited
from the Operations Center level in CloudBees CI on Kubernetes, or configured separately per
controller), pointing at a cluster with a node pool carrying the `workload: agent` label and
matching toleration. This guide intentionally does **not** fabricate a `clouds:` CasC snippet,
since it would require your cluster's API endpoint, namespace, credentials, and Jenkins URL —
details not present anywhere in `samples/` or `demo-jobs-casc/`. Confirm this cloud exists and is
healthy (**Manage Jenkins → Clouds**) before running any job beyond `00-HeloWorld`.

### 5.3 Credentials

| ID | Type | Value source | Used by |
| --- | --- | --- | --- |
| `gh-app` | GitHub App | `appID: ${GITHUB_APP_ID}`, `privateKey: ${GITHUB_APP_PRIVATE_KEY}`, `repositoryAccessStrategy: inferOwner` | `02-MultiBranchJob-HelloWorld`, `03-GitHub-Organisation`, `05-PTC-job-MB`, the global Template Catalog registration |
| `gh-pat` | Username/Password (scope GLOBAL) | `username: ${GITHUB_USERNAME}`, `password: ${GITHUB_PAT}` | `01-pipeline-simple` checkout, `template-catalog` checkout inside `02-MultiBranchJob-HelloWorld-customMarker` / `03-GitHub-Organisation`, and the shared-library dynamic `library()` step's default credential |
| `gh-pat-key` | Secret text (scope GLOBAL) | `secret: ${GITHUB_PAT}` | Classic GitHub plugin config (`unclassified.gitHubPluginConfig`) — used only for webhook management, not checkout |

> The shared library (`samples/shared-library`) is loaded **dynamically** via the `library()` step
> with coordinates defaulted inline in each Jenkinsfile (`SHAREDLIB_GIT_*` env vars defaulting to
> `pipeline-training-ws/shared-library@main` via credential `gh-pat`) — see
> `samples/sample-app-helloWorld/Jenkinsfile` and
> `samples/template-catalog/templates/1-helloWorld-MB/Jenkinsfile`. There is **no**
> `globalLibraries:` block in `jenkins.yaml`, so no separate "Global Pipeline Library"
> registration step is needed for this workshop's jobs.

### 5.4 Pipeline Template Catalog registration

```yaml
globalCloudBeesPipelineTemplateCatalog:
  catalogs:
  - branchOrTag: "main"
    parentName: "/"
    scm:
      github:
        credentialsId: "gh-app"
        repoOwner: "pipeline-training-ws"
        repository: "template-catalog"
    updateInterval: "1d"
```

This registers `samples/template-catalog` (its `catalog.yaml` names it **"Pipeline Template
Catalog Examples"**) globally, at the root (`parentName: "/"`), refreshed daily. Confirm it has
synced (**Manage Jenkins → Pipeline Template Catalogs**) before creating `05-PTC-job-MB` — that
job's `model:` field references an internally generated template ID that only resolves once the
catalog has synced at least once (see §8, job 8).

### 5.5 Other unclassified settings

| Block | Effect |
| --- | --- |
| `beekeeper` | Enables security-warning and plugin-upgrade advisories (`autoUpgradePlugins: true`, `autoDowngradePlugins: false`) |
| `buildDiscarders.configuredBuildDiscarders` | Global build discarder: honour per-job settings first (`jobBuildDiscarder`), then fall back to `numToKeepStr: "2"` |
| `cloudBeesSCMReporting.enabled` | Enables GitHub Checks-style SCM reporting used by the `cloudBeesSCMReporting` trait on branch sources |
| `cloudbeesPipelineExplorer` | Enables the Pipeline Explorer UI with auto-polling every 5s |
| `gitHubConfiguration.apiRateLimitChecker` | `ThrottleForNormalize` — throttles GitHub API calls instead of failing outright near rate limits |
| `gitHubPluginConfig` | Classic GitHub plugin webhook config; `hookUrl: https://${JENKINS_CONTROLLER_URL}/github-webhook/` — **requires `JENKINS_CONTROLLER_URL` to be a publicly reachable hostname** for GitHub to deliver webhooks |
| `location.url` | `https://${JENKINS_CONTROLLER_URL}` — the controller's externally visible base URL |

---

## 6. RBAC (`rbac.yaml`)

`removeStrategy.rbac: SYNC` — this bundle is authoritative for RBAC on reload; roles/bindings not
present here are removed.

| Role | Scope | Notable permissions |
| --- | --- | --- |
| `administer` | Global | Full admin: `Hudson.Administer`, credential CRUD, CasC admin, plugin uploads, etc. |
| `develop` | Filterable (folder-assignable) | Item build/create/configure/delete/read, run update/delete, view management — no controller-wide admin |
| `browse` | Filterable (folder-assignable) | Read-only: `Item.Read`, `Item.Discover`, `Hudson.Read`, `View.Read` |
| `authenticated`, `anonymous` | Built-in | No custom permissions attached beyond the platform default |

Folder-level binding: `RBAC-Folder-dev1-only` (in `items.yaml`) restricts itself to the `develop`
and `browse` roles and grants a group `folder-dev1` (containing user `dev1`) the `develop` role,
`grantedAt: current`. This is what §2 item 6 requires `dev1` to already exist for.

**Excluded from this guide:** `rbac.yaml` also defines a `serviceAccounts` entry named
`asahjdcfshjdfc` with a password-type authentication token (description `mytoken`, expiring
`2026-09-17T11:57:03Z`). It is not referenced by any job, credential, or role binding anywhere in
`demo-jobs-casc/` or `samples/`, and its name doesn't follow any naming convention used elsewhere
in the bundle — it looks like a leftover artifact from manual testing rather than an intentional
part of this workshop's setup. **This guide does not instruct recreating it.** If it *is*
intentional, let me know what it's for and I'll add proper setup steps.

---

## 7. Security Realm

`jenkins.yaml` sets `securityRealm: "operationsCenter"` — this controller does not define its own
user database, LDAP, SAML, or OAuth realm; it defers entirely to whatever realm the parent
Operations Center is configured with. Practical implication: to satisfy prerequisite #6 (user
`dev1` must exist), create/confirm that user in the **Operations Center's** security realm, not on
this controller — the specific steps depend on which realm type your Operations Center uses (this
is not discoverable from `samples/` or `demo-jobs-casc/`).

---

## 8. Jobs (`items.yaml`)

`removeStrategy: { rbac: SYNC, items: NONE }` — job removal is **not** synced; deleting an item
from `items.yaml` will not delete it from the controller on reload.

Set jobs up in this order — later jobs depend on credentials/catalog registrations established
earlier, and the grouping below mirrors the workshop's four topic segments.

### Segment 1 — Architecture & CasC Foundation (no external repo/credentials needed)

**1. `00-HeloWorld`** *(pipeline, inline script, `agent none`)*

- Fastest smoke test that `items.yaml` parsed and synced. No SCM, no agent.
- Apply the bundle, open the job, **Build Now**, confirm console output prints `Hello World`.

**2. `06-parallel-samples` folder → `agent-global`** *(pipeline, inline script)*

- Single Kubernetes pod (`maven` + `node` containers) shared by both parallel branches — the
  "monolith" pattern from `samples/pipeline-samples/parallel/Jenkinsfile-global-agent`.
- Requires the Kubernetes cloud from §5.2.
- Run and inspect the Kubernetes tab: one pod, two sibling containers.

**3. `06-parallel-samples` folder → `agent-per-stage`** *(pipeline, inline script)*

- `agent none` at top level; each parallel branch requests its own pod — the "distributed" pattern
  from `samples/pipeline-samples/parallel/Jenkinsfile-stage-agent`.
- Run and confirm **two separate pods**, contrasting with `agent-global`.

**4. `RBAC-Folder-dev1-only`** *(folder, RBAC-scoped)*

- Confirm `dev1` exists in the OC's security realm (§7) **before** reload — unknown-user bindings
  are silently ignored.
- Log in as `dev1`; verify this folder is visible with **Develop** permissions while other folders
  are not.

### Segment 2 — Shared Library Implementation

**5. `01-pipeline-simple`** *(pipeline, `cpsScmFlowDefinition`)*

- Points at `https://github.com/pipeline-training-ws/sample-app-helloworld.git`, `main` branch,
  `Jenkinsfile` at repo root, credential `gh-pat`.
- That Jenkinsfile dynamically loads `shared-library` and calls `pipelineTemplateHelloWorld`,
  which runs on a Kubernetes pod using `shared-library/resources/podtemplates/agent.yaml`.
- Requires the Kubernetes cloud from §5.2, since `pipelineTemplateHelloWorld` explicitly requests
  a `kubernetes` agent for its `CI` stage.
- Run and confirm `Load Config`, `Hello World`, and (on `main` only) `Hi` stages execute, printing
  values sourced from `sample-app-helloWorld/ci-config.yaml` (`hello: world`,
  `firstName: Andreas`, `lastName: Caternberg`).

### Segment 3 — Pipeline Types and Templates

All jobs in this segment read from `pipeline-training-ws` via the `gh-app` credential — confirm
the GitHub App installation covers `sample-app-helloWorld` and `template-catalog` first (§2 item
2).

**6. `02-MultiBranchJob-HelloWorld`** *(multibranch)*

- Standard Multibranch Pipeline scanning `sample-app-helloworld`, using the repo's own
  `Jenkinsfile` (`workflowBranchProjectFactory`), branch/PR/fork discovery traits, GitHub Checks
  reporting trait, `periodicFolderTrigger: 1h`.
- Save, **Scan Repository Now**, confirm a child job for `main` (and any open PRs/forks per
  discovery traits).

**7. `02-MultiBranchJob-HelloWorld-customMarker`** *(multibranch)*

- Same repo, but `customBranchProjectFactory` with `marker: ci-config.yaml`: only branches
  containing that file get a child job. The pipeline definition itself is loaded from
  `template-catalog`'s `templates/1-helloWorld-MB/Jenkinsfile` (checked out via `gh-pat`).
- Scan and confirm child jobs only appear for branches carrying `ci-config.yaml`. Remove/re-add
  that file on a test branch and re-scan to see the child job disappear/reappear.

**8. `03-GitHub-Organisation`** *(organizationFolder)*

- Scans the entire `pipeline-training-ws` org (`repoOwner`, credential `gh-app`), with two
  marker-based project factories:
  - `ci-config.yaml` → `template-catalog`'s `templates/1-helloWorld-MB/Jenkinsfile` (same pattern
    as job 7, applied org-wide).
  - `ci-config-1.yaml` → a minimal inline `echo 'Hello World'` pipeline.
- `periodicFolderTrigger: 1d`; save and let it run, or trigger **Scan Organization Folder Now**
  manually. Confirm repos/branches are picked up according to whichever marker file they contain.

**9. `05-PTC-job-MB`** *(`cloudbeesTemplatedJob`)*

- Instantiated from the **Pipeline Template Catalog Examples** catalog (§5.4), model
  `Pipeline-Tem.c3qk18.log-Examples/1-helloWorld-MB`, pre-filled with attributes
  `githubToken: gh-app`, `repoName: sample-app-helloWorld`, `organisation: pipeline-training-ws` —
  matching `template-catalog/templates/1-helloWorld-MB/template.yaml`'s declared parameters.
- **Confirm the catalog has synced first** (**Manage Jenkins → Pipeline Template Catalogs**) —
  otherwise the `model:` reference won't resolve. That ID is generated per-controller by
  CloudBees CI and **will drift** if the catalog is re-registered; if `items.yaml` fails to apply
  because of it, recreate this item by hand via **New Item → Pipeline Template Catalog Examples →
  Template 1 helloWorld-MB**, using the same three attribute values, then re-export.

### Segment 4 — CloudBees Pipeline Features

**10. `08-checkpoints`** *(pipeline, inline script)*

- Two Kubernetes-agent stages (`Say Hello`, `Say Good By`) separated by `checkpoint 'Hello'` and
  `checkpoint 'ByBy'`.
- Requires the Kubernetes cloud (§5.2) and the `workflow-cps-checkpoint` plugin.
- Run once to completion, then open the build → **Restart from Checkpoint → Hello** → confirm the
  new build skips straight to `Say Good By`.

**11. `07-crossteam-collaboration-events` folder** *(3 pipelines: `Event-producer`,
`Event-consumer-1`, `Event-consumer-2`)*

- `Event-producer` runs on a `maven` pod and calls `publishEvent` with a
  `com.example:supply-chain-webapp:1.2-SNAPSHOT` event.
- Both consumers run on `curl` pods and subscribe via
  `eventTrigger jmespathQuery("contains(event, 'com.example:') && contains(event, '-SNAPSHOT')")`.
- Requires the Kubernetes cloud (§5.2) and the `pipeline-event-step` plugin.
- **Create/enable all three jobs first** — the consumers' triggers must be registered before the
  event fires — then run `Event-producer` manually and confirm both consumers fire automatically.

---

## 9. Validation Checklist

- [ ] Bundle reload reports success on the bundle management page (§4)
- [ ] `00-HeloWorld` prints `Hello World` with no SCM/agent involved
- [ ] Kubernetes cloud is healthy and schedules pods on `workload: agent` nodes (§5.2)
- [ ] `gh-app`, `gh-pat`, `gh-pat-key` credentials resolve (no `${...}` left unresolved in
      **Manage Jenkins → Credentials**)
- [ ] Pipeline Template Catalog **Pipeline Template Catalog Examples** shows as synced
- [ ] `dev1` exists in the OC's security realm and sees only `RBAC-Folder-dev1-only` with Develop
      permissions
- [ ] `06-parallel-samples/agent-global` produces one pod with two containers;
      `agent-per-stage` produces two independent pods
- [ ] `01-pipeline-simple` prints config values from `ci-config.yaml` via the shared library
- [ ] Both Multibranch jobs (`02-*`) and the Org Folder (`03-GitHub-Organisation`) discover
      branches matching their respective marker-file rules
- [ ] `05-PTC-job-MB` runs successfully via the Template Catalog-instantiated job
- [ ] `08-checkpoints`: Restart from Checkpoint `Hello` skips `Say Hello` and resumes at
      `Say Good By`
- [ ] `07-crossteam-collaboration-events`: running `Event-producer` triggers both
      `Event-consumer-1` and `Event-consumer-2` automatically

---

## 10. Open Items Requiring Your Input

These were deliberately left as flagged gaps rather than filled in with assumptions:

1. **`labs/` directory** describes an unrelated environment (different org/repo/credential names)
   and was not reconciled with this guide — clarify whether it should be updated, replaced, or
   removed.
2. **Kubernetes cloud CasC** is not included; §5.2 documents it only as an external prerequisite.
   If you want it added to `jenkins.yaml`, provide the cluster API endpoint, namespace,
   credential, and Jenkins URL to use.
3. **The `asahjdcfshjdfc` service account** in `rbac.yaml` was excluded from the setup steps as an
   apparent test artifact (§6) — confirm whether it should be documented/kept or removed from the
   bundle.
4. **Environment-variable injection mechanism** for `GITHUB_APP_ID`, `GITHUB_APP_PRIVATE_KEY`,
   `GITHUB_PAT`, `GITHUB_USERNAME`, `JENKINS_CONTROLLER_URL` is referenced generically in §4 — the
   exact mechanism (Kubernetes Secret, OC bundle env-var UI, secrets directory, etc.) depends on
   how your specific controller is deployed and wasn't specified in `demo-jobs-casc/` or
   `samples/`.
5. **GitHub App private key files** (`converted-github-app.pem`,
   `pipeline-ws-2026.2026-08-14.private-key.pem`) committed at the project root — flagged as a
   possible secrets-handling concern, not something this guide resolves.
