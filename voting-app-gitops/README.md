# Voting-App in Kubernetes

Modern software systems rely heavily on microservices to achieve scalability, flexibility, and faster delivery cycles. However, building, testing, and deploying multiple microservices across different environments introduces significant operational complexity. To address these challenges, this project implements a complete DevOps pipeline that automates the build and testing of microservices, pushes container images to a Nexus registry, and standardizes deployments using Helm and Kustomize. By adopting GitOps principles with ArgoCD, the project ensures consistent, secure, and observable deployments across production, staging, and development environments. This approach not only streamlines operations but also provides a practical, hands-on framework for applying DevOps best practices in real-world scenarios.

In this project, we pursue these goals:

* Create a pipeline that builds our 3 main microservices, runs our desired tests, and pushes the built images to a Nexus registry.

* Create a Helm chart for the microservices based on their manifests.

* Use Kustomize to deploy microservices in different environments such as production, stage, and development.

* Also, deploying microservices should be done with GitOps, and for this purpose, we use ArgoCD.

It should be noted that for more information about the microservices themselves, you can refer to the original link of this project that I used.
https://github.com/dockersamples/example-voting-app

## Overview of the steps we take

### Pipeline
* Our architecture is microservice and all project components must be Dockerized.
* Developer changes to the application source code
* We need to control execution of pipeline with workflow rules:
  * If you are on the feature branch and don't have a Merge Request (MR) → nothing will be done.
  * If you have an MR to the development branch, the pipeline will be run.
  * If you have an MR to main → the pipeline will be run.
  * Otherwise → nothing will be done.
* Developer Commit changes and opens MR to `dev` or `main`.
* To control where pipeline commands are executed, we use tags. For this purpose, we must have tagged the runner when registering it.
* Only the build, test, and push stages are done in CI. The deploy stage wiil be done with ArgoCD.
* As much as possible, we turn all repeating items into variables.
* Build stages are only executed when files in the same service directory have changed and MR is set to dev or main. This helps prevent time-consuming and unnecessary builds.
* CI pipeline running (only Build & Push)
  * Build image
  * Push image into registry

#### Pipeline description   

workflow rules:\
We specified when the pipeline will be executed in general:

* ❌ If you are on the feature branch and have not given a Merge Request (MR) → nothing will be done.
* ✅ If you have a MR to development branch → the pipeline will be executed.
* ✅ If you have a MR to main branch → the pipeline will be executed.
* Otherwise → nothing will be executed.

That is, CI will be activated only when you have MR to `dev` or `main`.

### Build Stage

Example: build_vote

A Docker image is built for the vote service

It is only executed when the files in vote/**/* have changed and MR is `dev` or `main` → .

The same logic applies to result and worker.
👉 This means that only the service that has changed is built, not all services.

#### Test Stage Overview

Our CI/CD pipeline includes automated tests and code quality checks that run before building and pushing Docker images. These steps help us ensure that the code is correct, maintainable, and secure.

1 - test_vote (Python Unit Tests)

Runs automated unit tests for the vote service.

Installs the service’s dependencies (from requirements.txt or pyproject.toml).

Executes tests using `pytest` . Unit tests verify that small pieces of code (functions, modules) work as expected. This prevents bugs from going into production.

2 - sonar_vote (Static Code Analysis with SonarQube)

Analyzes the Python source code in the vote service.

Reports on code quality issues, bugs, security hotspots, and maintainability concerns.

SonarQube ensures the code follows good practices, is easy to maintain, and is free of common vulnerabilities. It’s like a spell-checker, but for code quality and security.

3 - trivy_vote (Security Scan with Trivy)

Scans the vote service for known vulnerabilities. Uses Trivy to scan the project’s files and dependencies.

Looks for vulnerabilities with High or Critical severity.

If vulnerabilities are found, they are reported so developers can fix them.

Trivy helps protect the service against security threats by detecting outdated or insecure libraries early in the development process.

#### Why Run These Tests Before Build?

By running unit tests, code analysis, and vulnerability scans first, we avoid building and publishing broken or insecure images. Only code that passes these checks continues to the build and deployment stages.

### Helm Charts

Below you’ll find a practical, technical, and actionable walkthrough to create Helm charts for microservices starting from raw Kubernetes manifests. I assume Helm 3 and kubectl are available locally and you have a working container registry.

#### 1 - Create a chart skeleton

From the repo root, run for each service (example for result):
```
cd helm/charts
helm create result
```

This produces the canonical structure:
```
helm/charts/result/
  Chart.yaml
  values.yaml
  templates/
    deployment.yaml
    service.yaml
    _helpers.tpl
    ...
```

If you prefer minimal manual skeletons, create only Chart.yaml, values.yaml, and templates/ and add files you need.

#### 2 - Design values.yaml (single source of environment variations)

Create a clear, opinionated values.yaml that holds all tunables. Example (shortened):
```
replicaCount: 1

image:
  repository: registry.example.ir/result
  tag: v1
  pullPolicy: IfNotPresent
  ...
```

Rules of thumb

* Keep values.yaml safe for dev (low replicas, permissive resources).
* Put secrets only as references (secret names) — do not store plaintext secrets in Git.
* Use image.tag to promote releases; avoid embedding SHA in chart itself — use CI to update tags or use ArgoCD image updater.

#### 3 - Convert manifests to Helm templates

Open templates/deployment.yaml and replace hardcoded fields with Helm template expressions.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: result
spec:
  replicas: {{ .Values.replicaCount }}
  ...
```

#### 4 - Lint, render, and test locally

Validate the chart:
```
helm lint helm/charts/result

# Render templates with the default values
helm template result helm/charts/result --namespace result

# Render with a custom values file (eg. values-prod.yaml)
helm template result helm/charts/result -f helm/charts/result/values-prod.yaml --namespace result

# Dry run install to test rendering logic:
helm upgrade --install result-test helm/charts/result --namespace result --create-namespace --dry-run --debug
```

Use `kubeval` or `kubectl apply --dry-run=client` to further validate.

#### 5 - Packaging & publishing charts

You have two options:

A. Host charts in Git (Helm chart lives in repo) and let ArgoCD use repo path (simplest for GitOps).

B. Package and publish to a Helm repo (ChartMuseum, Nexus/Helm hosted repo):

```
# package
helm package helm/charts/result -d ./packages

# push (depends on your repo server)
# Option 1: use helm-push plugin:
helm repo add my-charts https://charts.mycompany.com
helm push ./packages/result-0.1.0.tgz my-charts
```


Let’s take a step-by-step look at exactly what this Helm template does when executed and what each line means. \
The `helm/charts/db/templates/db.yaml` file creates two Kubernetes objects: 
* a Deployment for PostgreSQL and
* a Service to access it.

<img width="855" height="810" alt="image" src="https://github.com/user-attachments/assets/0db02702-c708-4d73-9874-4be206d5c644" />


The braces `{{ ... }}` are helm variables and are filled with `values.yaml` (or --set):
* `.Values.replicaCount` → number of pods
* `.Values.image.repository`, `.Values.image.tag`, `.Values.image.pullPolicy` → postgres image
* `.Values.service.port` → Postgres port (usually 5432)
* `.Values.service.type` → service type (default ClusterIP)
* `.Values.env.POSTGRES_USER`, `.Values.env.POSTGRES_PASSWORD` → user and password for postgresql database
When you run `helm install` or `helm upgrade`, this file is converted to the final YAML and sent to the Kubernetes server API.

### Kustomize

Kustomize is a Kubernetes-native configuration management tool that allows you to customize Kubernetes YAML manifests without modifying the original files. Instead of maintaining separate manifest copies for each environment (development, staging, production), Kustomize enables you to define a common base and apply overlays (environment-specific adjustments) on top of it.

Key benefits:

* Eliminates duplication: Reuse the same base resources across environments.
* Declarative: Everything is expressed as YAML, no templating language required.
* Supports Helm charts: Kustomize can consume Helm charts directly, making it possible to unify Helm templating and Kustomize overlays.

Each environment (e.g., dev or prod) has its own kustomization.yaml file that defines which resources to build and how to customize them.

Instead of applying raw Helm charts or raw YAML manifests, Kustomize does the following:

1 - Reads the Helm charts from helm/charts/.

2 - Applies the environment-specific values files (e.g., values-result.yaml, values-db.yaml).

3 - Builds a set of manifests tailored for the given namespace and environment.

4 - Produces a final bundle of Kubernetes manifests ready to be applied by ArgoCD (or manually with kubectl apply -k).

This approach keeps your environment configuration organized, repeatable, and GitOps-friendly.

#### Operational considerations

* Each environment (dev, prod, stage) should have its own directory with a `kustomization.yaml`and environment-specific values files.
* Example: k8s-specifications/prod/values-result.yaml may set replicaCount=5 and a stronger resource limit, while dev may set replicaCount=1.
* Avoid committing plain secrets in `values-*.yaml`. Use external secret managers (Vault or ...) and reference them from your charts instead.
* ArgoCD can point directly to k8s-specifications/dev or k8s-specifications/prod.
* ArgoCD will run Kustomize automatically, apply the Helm charts with your values, and sync the resources.

Kustomize ⇒ A way to reuse manifests between different environments
```
helmCharts:
  - name: db
    path: ../../helm/charts/db
    valuesFile: values-db.yaml
```
This means:\
"Use this Helm Chart, create actual Kubernetes files with these values ​​(values-db.yaml)."

valuesFile → a YAML file next to kustomization.yaml 
That means values-db.yaml should be here:
```
k8s-specifications/prod/
├── kustomization.yaml
└── values-db.yaml 
```
Why here?\
Because valuesFile only says the file name (values-db.yaml), not the full path.\
So Kustomize looks for that file in the same directory as kustomization.yaml.\

For each service (db, redis, vote, result, worker) you should have a values ​​file in the same prod (or dev) directory:
```
k8s-specifications/prod/
├── kustomization.yaml
├── values-db.yaml
├── values-redis.yaml
├── values-vote.yaml
├── values-result.yaml
└── values-worker.yaml
```

### ArgoCD and GitOps

This repository uses Argo CD and its ApplicationSet feature to manage Kubernetes deployments using the GitOps approach.

What is GitOps?

GitOps means treating Git as the single source of truth for your infrastructure and application configuration.

Instead of manually applying YAML files to Kubernetes, you store them in a Git repository.

Argo CD continuously watches the repository and makes sure the Kubernetes cluster matches what’s defined in Git.

  * ArgoCD does not perform Build task.
  * ArgoCD also does not perform the Tagging task.
  * ArgoCD just looks at what version is written in Git and deploys that.
  * CI should update `tag` value in these files: `values-vote.yaml` or `values-result.yaml`
  * Why do we change these, and not that `helm/charts/vote/values.yaml`?
  * Because ArgoCD is already looking at the path `k8s-specifications/dev/` (according to ApplicationSet).
  * If we change this → ArgoCD syncs → deployment is updated.
  * Image Updater do these changes

What is an ApplicationSet?
* Normally, Argo CD manages one application per configuration file.
* The ApplicationSet controller makes it possible to define multiple applications at once in a single template.
* This is very useful when you want to deploy the same app into multiple environments (e.g., dev, prod) with just one configuration.

#### How This ApplicationSet Works

In our case, the ApplicationSet is named voting-environments. It defines how to deploy the Voting App into both `development` and `production` environments.

Generators

The generator section lists environments (dev, prod) along with their:
- path: Where the Kubernetes YAML manifests are stored in the Git repo
- server: The Kubernetes cluster API address
- namespace: The target namespace in the cluster

Template
- The template tells Argo CD how to create each application.
- It uses placeholders like `{{env}}`, `{{path}}`, and `{{namespace}}` that get replaced for each environment defined in the generator.

For example:

dev-voting-app → points to the dev manifests and deploys into the voting-dev namespace.

prod-voting-app → points to the prod manifests and deploys into the voting-prod namespace.

Sync Policy

automated: {} means Argo CD will automatically apply changes from Git to the cluster without requiring manual approval.

This ensures both clusters stay continuously in sync with the repository.

ArgoCD reads this ApplicationSet.
Here we have two environments:\
* dev → on the internal cluster (https://kubernetes.default.svc) and namespace=voting-dev
* prod → on the external cluster (https://10.20.20.31:6443) and namespace=voting-prod

So ArgoCD creates two Applications:
dev-voting-app
prod-voting-app

### ArgoCD Image Updater

ArgoCD itself is installed on the cluster (in  argocd namespace).

Image Updater is a separate service that can run on the same kubernetes cluster.

Installation is usually done with Helm or kubectl:
```
kubectl create namespace argocd-image-updater
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd-image-updater argo/argocd-image-updater -n argocd-image-updater
```
This will create a deployment for Image Updater on the cluster.

* Image Updater needs to know which `applications` and which `images` to control.
* One way is to add some `annotations` to the `Application` manifests and configure the 'Image Updater' with them.
Example:
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-set-for-vote-result
  namespace: argocd
  annotations:
    argocd-image-updater.argoproj.io/image-list: |
      vote=docker.rahbia.iamsre.ir/vote
      result=docker.rahbia.iamsre.ir/result
    argocd-image-updater.argoproj.io/vote.update-strategy: latest
    argocd-image-updater.argoproj.io/result.update-strategy: semver:minor
```
latest → use any new tag released \
semver:minor → only update when the minor version changes
 

  
* Image Updater should be able to modify manifest or helm `values.yaml` files and commit/push.
