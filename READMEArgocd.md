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


