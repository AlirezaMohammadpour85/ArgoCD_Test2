### Install Argo CD inside the Kubernetes cluster, meaning inside your kind cluster running in the VM.

Not on your Mac. Not directly on the VM OS.
Argo CD is a Kubernetes application, so it runs as Pods/Services/Deployments inside the cluster.

Your setup should be like this:

Mac
 ├── kubectl
 ├── helm
 └── kubeconfig points to VM kind cluster

VM
 └── kind Kubernetes cluster
      ├── your test apps
      ├── your Helm chart releases
      └── argocd namespace
           ├── argocd-server
           ├── argocd-repo-server
           ├── argocd-application-controller
           └── argocd-dex / redis / etc.

So from your Mac terminal, because your kubectl already works, you can install Argo CD into the VM cluster like this:

kubectl create namespace argocd

Then install Argo CD:

kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

Check it:

kubectl get pods -n argocd

You should see Argo CD pods running inside the kind cluster.

To access the Argo CD UI from your Mac, use port-forward:

kubectl port-forward svc/argocd-server -n argocd 8080:443

Then open:

https://localhost:8080

Get the initial admin password:

kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d

Login:

username: admin
password: the password from the command

Important idea:

kubectl and helm can be on your Mac.
Argo CD itself must be installed inside the Kubernetes cluster.

For your lab, this is the best architecture:

Mac = control machine
VM = Kubernetes host
kind = Kubernetes cluster
Argo CD = installed inside kind

Later, when you work with real production clusters, Argo CD is also usually installed inside a Kubernetes cluster, often called the management cluster, and then it deploys apps to the same cluster or to other clusters.

### Now that Argo CD is installed, the next goal is: access the UI/CLI, then create your first Argo CD Application that deploys from a Git repo into your kind cluster.

Thought for a few seconds

Yes. Since Argo CD is installed inside the same kind cluster, it can already manage that cluster. You do not need to “connect” the same cluster manually.

In Argo CD, the in-cluster Kubernetes cluster is usually available as:

https://kubernetes.default.svc

That means:

Argo CD is inside kind
→ it uses its ServiceAccount
→ it can deploy apps into the same kind cluster
Next steps
1. Check Argo CD pods
kubectl get pods -n argocd

Wait until the main pods are Running:

argocd-server
argocd-repo-server
argocd-application-controller
argocd-redis
argocd-dex-server
2. Access Argo CD UI

Run this from your Mac, because your Mac kubectl already connects to the VM kind cluster:

kubectl port-forward svc/argocd-server -n argocd 8080:443

Then open:

https://localhost:8080

Your browser may show a certificate warning. For local lab, that is okay.

3. Get initial admin password

Current Argo CD docs recommend this:

argocd admin initial-password -n argocd

You can also get it with kubectl:

kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

The username is:

admin

Argo CD stores the initial admin password in argocd-initial-admin-secret; after changing the password, the docs recommend deleting that initial secret.

4. Install Argo CD CLI on Mac

On macOS:

brew install argocd

Then login:

argocd login localhost:8080

For local port-forward with self-signed cert, use:

argocd login localhost:8080 --insecure

The Argo CD CLI login command is used to log in to an Argo CD API server with username/password or SSO.

5. Create your first app

You have two choices:

Option A — use Argo CD UI

In the UI:

New App

Example values:

Application Name: demo-app
Project: default
Sync Policy: Manual

Repository URL: your GitHub repo URL
Revision: main
Path: path/to/chart-or-yaml

Cluster URL: https://kubernetes.default.svc
Namespace: default

Then click:

Create → Sync
Option B — use CLI

Example with plain Kubernetes YAML from a Git repo:

argocd app create demo-app \
  --repo https://github.com/YOUR-USER/YOUR-REPO.git \
  --path k8s \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default

Then sync it:

argocd app sync demo-app

Check status:

argocd app get demo-app
For your Helm chart

Because you are already learning Helm, this is the best next test.

Put your Helm chart in Git, for example:

my-git-repo/
└── my-first-chart/
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
        ├── deployment.yaml
        └── service.yaml

Then create Argo CD app:

argocd app create my-helm-app \
  --repo https://github.com/YOUR-USER/YOUR-REPO.git \
  --path my-first-chart \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace helm-lab

Create namespace if needed:

kubectl create namespace helm-lab

Then sync:

argocd app sync my-helm-app

Check:

kubectl get all -n helm-lab
Important concept

Before Argo CD, you were doing this:

Mac terminal → kubectl/helm → kind cluster

With Argo CD, the flow becomes:

Git repo → Argo CD → kind cluster

So you should stop thinking like:

I run helm install manually

And start thinking like:

I push Helm chart/YAML to Git
Argo CD reads Git
Argo CD applies it to Kubernetes

For your current lab, the next best practice is:

1. Put your Helm chart into GitHub
2. Connect Argo CD to that repo
3. Create an Argo CD Application
4. Sync it
5. Change values.yaml
6. Push to Git
7. Watch Argo CD detect OutOfSync
8. Sync again



### Perfect. That is exactly the right next step.

For GitOps with Argo CD, the desired state should live in Git:

Git repository
  └── Helm chart
      ├── Chart.yaml
      ├── values.yaml
      ├── values-prod.yaml
      └── templates/
          ├── configmap.yaml
          ├── deployment.yaml
          └── service.yaml

Then Argo CD will watch that repo and apply changes to the cluster.

Answer to previous question

Correct GitOps answer is:

B. In Git, for example inside values-prod.yaml

Not in --set, not in kubectl edit, and not only in Helm history.

Lesson 19 — Prepare the Git repository

Create a new GitHub repository, for example:

helm-argocd-course

Public is easiest for the lab. Private also works, but then we need to configure repo credentials in Argo CD.

Recommended repo structure

Inside your local project, prepare this structure:

helm-argocd-course/
└── charts/
    └── my-first-chart/
        ├── Chart.yaml
        ├── values.yaml
        ├── values-prod.yaml
        └── templates/
            ├── configmap.yml
            ├── deployment.yaml
            └── service.yaml

Your current chart directory is:

my-first-chart-lesson11

So you can do:

mkdir -p ~/helm-argocd-course/charts

cp -r /path/to/my-first-chart-lesson11 ~/helm-argocd-course/charts/my-first-chart

cd ~/helm-argocd-course

Then initialize Git:

git init
git add .
git commit -m "Add first Helm chart"

Connect to GitHub:

git branch -M main
git remote add origin https://github.com/YOUR-USERNAME/helm-argocd-course.git
git push -u origin main

Replace:

YOUR-USERNAME

with your GitHub username.

Before pushing, check this

From inside the repo root, run:

tree

Expected:

.
└── charts
    └── my-first-chart
        ├── Chart.yaml
        ├── values.yaml
        ├── values-prod.yaml
        └── templates
            ├── configmap.yml
            ├── deployment.yaml
            └── service.yaml

Also run:

helm template demo charts/my-first-chart -f charts/my-first-chart/values-prod.yaml

This confirms Argo CD will also be able to render it later.

Your task now

Create the GitHub repo and push the chart.

Send me:

tree
git remote -v
helm template demo charts/my-first-chart -f charts/my-first-chart/values-prod.yaml

After that, we will create the Argo CD Application manifest.


# Note
Argo CD does not automatically create an app for every folder unless you use ApplicationSet. Normally, you create an Argo CD Application that points to one folder/path.

Simple production idea

Example Git repo:

gitops-repo/
├── apps/
│   ├── frontend/
│   │   ├── Chart.yaml
│   │   ├── values-dev.yaml
│   │   ├── values-prod.yaml
│   │   └── templates/
│   ├── backend/
│   │   ├── Chart.yaml
│   │   ├── values-dev.yaml
│   │   ├── values-prod.yaml
│   │   └── templates/
│   └── worker/
│       ├── Chart.yaml
│       ├── values-dev.yaml
│       ├── values-prod.yaml
│       └── templates/
└── argocd-apps/
    ├── frontend-dev.yaml
    ├── frontend-prod.yaml
    ├── backend-dev.yaml
    └── backend-prod.yaml

Each Argo CD Application points to one path, for example:

source:
  repoURL: https://github.com/your-user/gitops-repo.git
  targetRevision: main
  path: apps/frontend
  helm:
    valueFiles:
      - values-prod.yaml

So this Argo CD app manages only:

apps/frontend

Another Argo CD app could manage:

apps/backend
For your current lab

Your repo has only one chart:

my-first-chart-lesson11/
├── Chart.yaml
├── values.yaml
├── values-prod.yaml
└── templates/

So this is fine. Argo CD can point directly to the repo root:

path: .

Later, when you have multiple projects, use folders like:

charts/frontend
charts/backend
charts/worker

Then Argo CD can manage them separately.

Important mental model
One Argo CD Application = one desired application state

That usually means:

one app + one environment

For example:

frontend-dev
frontend-prod
backend-dev
backend-prod

So yes, in real production, you usually separate projects and environments clearly in Git. For now, keep one chart at repo root so we can learn the Argo CD flow simply.

### reat — Argo CD is installed correctly. All Argo CD Pods are running:

argocd-server
argocd-repo-server
argocd-application-controller
argocd-applicationset-controller
argocd-redis
argocd-dex-server
argocd-notifications-controller

Your Application was created:

my-first-chart   Unknown   Healthy

Now we need to understand why SYNC STATUS is Unknown.

Healthy means the Argo CD Application object itself exists and is valid enough to be evaluated.
Unknown usually means Argo CD has not successfully compared Git desired state with cluster live state yet. Common reasons:

repoURL wrong
repo is private and Argo CD has no credentials
path wrong
branch wrong
Helm rendering error
Argo CD has not refreshed yet
Lesson 20 — Debug Argo CD Application status

Run this:

kubectl describe app my-first-chart -n argocd

Also run:

kubectl get app my-first-chart -n argocd -o yaml

But first, try a manual refresh by annotating the app:

kubectl annotate app my-first-chart -n argocd \
  argocd.argoproj.io/refresh=hard \
  --overwrite

Then check again:

kubectl get app my-first-chart -n argocd
Also check the destination namespace
kubectl get ns helm-lab
kubectl get all -n helm-lab
About your expected answers
Will Argo CD run helm install demo . exactly like your terminal did?
No. Argo CD does not manage it as a normal Helm CLI release. It uses Helm to render templates, then Argo CD applies the generated manifests.
Which namespace contains the Argo CD Application object?
argocd
Which namespace contains nginx Deployment and Service?
helm-lab
Which Git folder does Argo CD use?
charts/my-first-chart-l18

Send me the output of:

kubectl describe app my-first-chart -n argocd
kubectl get app my-first-chart -n argocd -o yaml

### Lesson 21 — Refresh and sync the Argo CD app

First force Argo CD to refresh the Git repo:

kubectl annotate app my-first-chart -n argocd \
  argocd.argoproj.io/refresh=hard \
  --overwrite

Then check:

kubectl get app my-first-chart -n argocd

Expected status should change from:

SYNC STATUS   HEALTH STATUS
Unknown       Healthy

to something like:

SYNC STATUS   HEALTH STATUS
OutOfSync     Healthy

or:

Synced        Healthy
If it becomes OutOfSync

That means Argo CD can now read Git and compare desired state with the cluster, but it has not applied the Git state yet.

Run:

kubectl patch app my-first-chart -n argocd --type merge -p '{
  "spec": {
    "syncPolicy": {
      "automated": {
        "prune": true,
        "selfHeal": true
      },
      "syncOptions": [
        "CreateNamespace=true"
      ]
    }
  }
}'

Then refresh again:

kubectl annotate app my-first-chart -n argocd \
  argocd.argoproj.io/refresh=hard \
  --overwrite

Check:

kubectl get app my-first-chart -n argocd
kubectl get all -n helm-lab
What we are testing

Now Argo CD should do this:

GitHub public repo
      ↓
Argo CD repo-server clones repo
      ↓
Argo CD renders Helm chart from charts/my-first-chart-l18
      ↓
Uses values-prod.yaml
      ↓
Applies resources into helm-lab

Send me the output of:

kubectl get app my-first-chart -n argocd
kubectl describe app my-first-chart -n argocd | sed -n '/Status:/,$p'


### so, in the concept of using argocd , we generally will not use helm alot, we only use the helm feature as areplacing the jinja2 templating or GO templating to be abkle use the vars inside the git and then argo will apply the changes

Yes — your understanding is mostly correct.

In an Argo CD GitOps workflow, you usually do not use Helm from your terminal every day to deploy changes.

Instead, Helm becomes the manifest generator / template engine.

Your new mental model

Before Argo CD:

helm upgrade demo . -f values-prod.yaml
        ↓
Helm talks directly to Kubernetes
        ↓
Cluster changes

With Argo CD:

Git repo
  ├── Chart.yaml
  ├── templates/
  └── values-prod.yaml
        ↓
Argo CD reads Git
        ↓
Argo CD runs Helm template/render internally
        ↓
Argo CD applies generated YAML to Kubernetes

So yes:

Helm = templating + variables + packaging structure
Argo CD = sync engine + GitOps controller + deployment manager
Important correction

Helm is not exactly “replacing Jinja2.” It is using Go templating, not Jinja2.

So in your case:

replicas: {{ .Values.replicaCount }}
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

Helm renders that into normal Kubernetes YAML:

replicas: 4
image: "nginx:1.27-alpine"

Then Argo CD applies it.

What you still use Helm for

Even with Argo CD, you still use Helm locally for testing:

helm template demo charts/my-first-chart-l18 -f charts/my-first-chart-l18/values-prod.yaml
helm lint charts/my-first-chart-l18

But you usually do not use this for deployment anymore:

helm upgrade demo ...

Because deployment is now Argo CD’s job.

Production workflow

A normal change becomes:

1. Edit values-prod.yaml
2. git add / commit / push
3. Argo CD detects OutOfSync
4. Argo CD syncs manually or automatically
5. Kubernetes matches Git

Example:

replicaCount: 4
image:
  tag: "1.28-alpine"

After push, Argo CD renders the Helm chart and applies the changed Deployment.

Very important GitOps rule

Avoid doing this after Argo CD manages the app:

helm upgrade demo ...
kubectl edit deployment demo ...
kubectl scale deployment demo ...

Because those are manual cluster changes. Argo CD may mark the app as OutOfSync, and if self-heal is enabled, it will revert the cluster back to Git.

So your final concept is:

Git is the source of truth.
Helm helps generate Kubernetes YAML from variables.
Argo CD makes the cluster match Git.

### Lesson 22 — Manual cluster change vs Git desired state

Argo CD’s job is to compare:

Desired state = Git
Live state    = Kubernetes cluster

If they differ, the app becomes:

OutOfSync
Current situation

Your Git has something like:

replicaCount: 4

or maybe 3, depending on your last pushed values-prod.yaml.

Your cluster should match that after Argo CD sync.

First check the current state:

kubectl get app my-first-chart -n argocd
kubectl get deployment demo -n helm-lab

Now we will intentionally create drift.

Step 1 — Manually scale the Deployment

Run:

kubectl scale deployment demo -n helm-lab --replicas=1

Then check:

kubectl get deployment demo -n helm-lab
kubectl get pods -n helm-lab
kubectl get app my-first-chart -n argocd

At first, Argo CD may still show Synced until it refreshes. Force refresh:

kubectl annotate app my-first-chart -n argocd \
  argocd.argoproj.io/refresh=hard \
  --overwrite

Then check again:

kubectl get app my-first-chart -n argocd
Prediction before running

Answer these first:

After kubectl scale --replicas=1, how many Pods will Kubernetes try to keep?
Will Git change?
Will Argo CD eventually see the app as OutOfSync?
If you click Sync in Argo CD GUI, what replica count will it restore?

This lesson teaches the core GitOps rule:

Manual cluster changes are temporary.
Git is the source of truth.


### Lesson 23 — Enable automatic sync and self-heal

Right now, your app probably requires manual sync from the GUI.

Now we will make Argo CD automatically apply Git changes.

Current behavior

Without automated sync:

Git changes
   ↓
Argo CD sees OutOfSync
   ↓
You manually click Sync
   ↓
Cluster updates

With automated sync:

Git changes
   ↓
Argo CD sees OutOfSync
   ↓
Argo CD syncs automatically
   ↓
Cluster updates

With self-heal:

Someone manually changes cluster
   ↓
Argo CD detects drift
   ↓
Argo CD restores Git state
Update your Argo CD Application

Edit your Application YAML:

nano argocd-apps/my-first-chart-app.yaml

Change syncPolicy to this:

syncPolicy:
  automated:
    prune: true
    selfHeal: true
  syncOptions:
    - CreateNamespace=true

So the full important part becomes:

spec:
  source:
    repoURL: https://github.com/AlirezaMohammadpour85/ArgoCD_Test2.git
    targetRevision: main
    path: charts/my-first-chart-l18
    helm:
      releaseName: demo
      valueFiles:
        - values-prod.yaml

  destination:
    server: https://kubernetes.default.svc
    namespace: helm-lab

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true

Apply it:

kubectl apply -f argocd-apps/my-first-chart-app.yaml

Check:

kubectl get app my-first-chart -n argocd -o yaml | grep -A8 syncPolicy
What do prune and selfHeal mean?

selfHeal: true

If someone changes the live cluster manually, Argo CD changes it back to Git.

Example:

kubectl scale deployment demo -n helm-lab --replicas=1

If Git says:

replicaCount: 4

Argo CD will restore it to 4.

prune: true

If you remove a resource from Git, Argo CD deletes it from the cluster.

Example: if you delete templates/configmap.yml from Git and push, Argo CD will delete:

configmap/demo-config

from Kubernetes.

Prediction

Before testing, answer these:

If Git says replicaCount: 4 and you manually scale to 1, what should Argo CD do?
If you delete service.yaml from Git and push, what should Argo CD do because prune: true is enabled?
Why is selfHeal useful in production?
Why can prune be dangerous if you delete the wrong file from Git?


### Lesson 24 — Test self-heal

Now apply the updated Application manifest:

kubectl apply -f argocd-apps/my-first-chart-app.yaml

Check that automated sync is enabled:

kubectl get app my-first-chart -n argocd -o yaml | grep -A8 syncPolicy

You should see:

syncPolicy:
  automated:
    prune: true
    selfHeal: true
  syncOptions:
  - CreateNamespace=true

Now create drift manually:

kubectl scale deployment demo -n helm-lab --replicas=1

Check immediately:

kubectl get deployment demo -n helm-lab
kubectl get pods -n helm-lab

Then refresh Argo CD:

kubectl annotate app my-first-chart -n argocd \
  argocd.argoproj.io/refresh=hard \
  --overwrite

Check again:

kubectl get deployment demo -n helm-lab
kubectl get app my-first-chart -n argocd
What should happen

First, Kubernetes accepts your manual change:

replicas: 1

Then Argo CD detects that live state differs from Git and restores it back to the replica count in values-prod.yaml.

So if Git says:

replicaCount: 4

the Deployment should return to:

4/4
Prediction before running

Answer this first:

When you run:

kubectl scale deployment demo -n helm-lab --replicas=1

will the command itself fail because Argo CD controls the app, or will it succeed and then Argo CD later correct it?