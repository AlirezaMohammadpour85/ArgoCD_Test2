### Lesson 11 — Add a Deployment template

We will deploy nginx using Helm variables.

Step 1: update values.yaml

Replace your values.yaml with this:

app:
  environment: default
  message: Hello from default nested values

replicaCount: 1

image:
  repository: nginx
  tag: "1.27"
  pullPolicy: IfNotPresent

containerPort: 80
Step 2: update values-prod.yaml

Replace your values-prod.yaml with this:

app:
  environment: production
  message: Hello from production nested values

replicaCount: 3

image:
  repository: nginx
  tag: "1.27-alpine"
  pullPolicy: IfNotPresent

containerPort: 80
Step 3: create templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
  labels:
    app: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: {{ .Release.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.containerPort }}
          env:
            - name: APP_ENVIRONMENT
              value: {{ .Values.app.environment | quote }}
            - name: APP_MESSAGE
              value: {{ .Values.app.message | quote }}

Keep your ConfigMap too. Now the chart should render two resources:

ConfigMap
Deployment
Prediction before running

For this command:

helm template demo . -f values-prod.yaml

Answer these:

What will the Deployment name be?
How many replicas will it have?
What image will the container use?
What value will APP_ENVIRONMENT have?

Then run:

helm template demo . -f values-prod.yaml


### Lesson 12 — Add a Service

Right now, the Deployment creates nginx Pods, but there is no stable internal access point for them.

A Kubernetes Service gives a stable name and IP for the Pods.

Step 1: add service values

Add this to both values.yaml and values-prod.yaml:

service:
  type: ClusterIP
  port: 80

So your values-prod.yaml should look like this:

app:
  environment: production
  message: Hello from production nested values

replicaCount: 3

image:
  repository: nginx
  tag: "1.27-alpine"
  pullPolicy: IfNotPresent

containerPort: 80

service:
  type: ClusterIP
  port: 80
Step 2: create templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
  labels:
    app: {{ .Release.Name }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app: {{ .Release.Name }}
  ports:
    - name: http
      port: {{ .Values.service.port }}
      targetPort: {{ .Values.containerPort }}

Important part:

selector:
  app: {{ .Release.Name }}

This Service will send traffic to Pods that have:

labels:
  app: demo

Your Deployment creates Pods with exactly this label:

template:
  metadata:
    labels:
      app: demo

So the Service can find the Pods.

Prediction

For:

helm template demo . -f values-prod.yaml

answer:

What will the Service name be?
What type will it have?
What port will it expose?
Which Pods will it select?

Then run:

helm template demo . -f values-prod.yaml

Send me your predictions and the rendered Service part.

### Lesson 13 — Upgrade the real release and test nginx

Now apply the new Deployment + Service to the existing Helm release:

helm upgrade demo . \
  -n helm-lab \
  -f values-prod.yaml

Then check the release:

helm history demo -n helm-lab

Check Kubernetes resources:

kubectl get all -n helm-lab

You should now see:

configmap/demo-config
service/demo
deployment.apps/demo
replicaset.apps/...
pod/demo-...
pod/demo-...
pod/demo-...

Because replicaCount is 3, you should get 3 Pods.

Then check if the Service found the Pods:

kubectl get endpoints demo -n helm-lab

or newer command:

kubectl get endpointslice -n helm-lab

If the Service selector is correct, endpoints should exist.

Test nginx

Run:

kubectl port-forward svc/demo 8080:80 -n helm-lab

Keep that terminal open.

In another terminal, run:

curl localhost:8080

You should see the nginx welcome HTML.

Prediction before running

Answer these first:

What new Helm revision number do you expect after this upgrade?
How many Pods should be created?
Why should the Service endpoints not be empty?
What will curl localhost:8080 return approximately?

Then run the commands and send me the outputs.

# NOTE
Your predictions are correct:

New revision should be 6.
Deployment should create 3 Pods.
Service endpoints should not be empty because selector: app=demo matches the Pods’ label app=demo.
curl localhost:8080 should return the nginx welcome page HTML.

Now your question:

Where does kubectl port-forward expose the port?

It exposes the port on the machine where you run kubectl.

So if you run this from your Mac:

kubectl port-forward svc/demo 8080:80 -n helm-lab

Then the port is opened on your Mac, not on the VM and not on the Kubernetes worker node.

You access it from your Mac with:

curl localhost:8080

or in your Mac browser:

http://localhost:8080
Mental model
Your Mac
kubectl port-forward
localhost:8080
      ↓
Kubernetes API server
      ↓
Service demo port 80
      ↓
One of the nginx Pods

The traffic tunnel goes through the Kubernetes API connection. That is why you do not need the Pod IP or NodePort.

Important detail

By default, kubectl port-forward listens only on your local machine:

127.0.0.1:8080 on your Mac

So another computer on your network cannot access it.

If you wanted to expose it on your Mac’s network interface, you could use:

kubectl port-forward --address 0.0.0.0 svc/demo 8080:80 -n helm-lab

But for learning and testing, use the safer default:

kubectl port-forward svc/demo 8080:80 -n helm-lab
Now continue the lab

Run:

helm upgrade demo . \
  -n helm-lab \
  -f values-prod.yaml

Then:

kubectl get all -n helm-lab
kubectl get endpoints demo -n helm-lab
kubectl port-forward svc/demo 8080:80 -n helm-lab

In another terminal:

curl localhost:8080

Send me the outputs of:

kubectl get all -n helm-lab
kubectl get endpoints demo -n helm-lab

### Lesson 15 — Scaling with Helm values

Now we will change the number of replicas through Helm.

Currently:

replicaCount: 3

in values-prod.yaml.

Instead of editing the file, use --set:

helm upgrade demo . \
  -n helm-lab \
  -f values-prod.yaml \
  --set replicaCount=5

Then check:

kubectl get pods -n helm-lab
kubectl get deployment demo -n helm-lab
kubectl get endpointslice -n helm-lab -l kubernetes.io/service-name=demo
helm get values demo -n helm-lab
helm history demo -n helm-lab
Prediction before running

Answer these first:

How many Pods should you see after the upgrade?
What should helm get values show for replicaCount?
Will this create a new Helm revision?
What should happen to the Service endpoints?

### Lesson 16 — Change the image tag with Helm

Now we will update the nginx image tag using Helm.

Currently your Pods are using:

nginx:1.27-alpine

We will change it to:

nginx:1.28-alpine

Run this:

helm upgrade demo . \
  -n helm-lab \
  -f values-prod.yaml \
  --set replicaCount=5 \
  --set image.tag=1.28-alpine

Then check:

kubectl get pods -n helm-lab
kubectl get deployment demo -n helm-lab -o yaml | grep image:
helm history demo -n helm-lab
helm get values demo -n helm-lab
Prediction before running

Answer these first:

Will Kubernetes create a new ReplicaSet?
Will all Pods be recreated?
How many Pods should be running after the upgrade?
What image should the Deployment show?
What new Helm revision number do you expect?

### Lesson 17 — Roll back the image update

Now let’s prove rollback affects the Deployment too.

Run:

helm rollback demo 7 -n helm-lab

Revision 7 was before the image changed to 1.28-alpine, based on your history.

Then check:

kubectl get pods -n helm-lab
kubectl get rs -n helm-lab
kubectl get deployment demo -n helm-lab -o yaml | grep image:
helm history demo -n helm-lab
helm get values demo -n helm-lab
Prediction

Before running, answer:

What new Helm revision will be created?
What image should the Deployment use after rollback?
Will Kubernetes create or reuse a ReplicaSet?
How many Pods should be running after rollback?


### Lesson 18 — Prepare for Argo CD transition

Before we move to Argo CD, we need to understand one important GitOps idea:

With normal Helm CLI you do this:

helm upgrade demo . -n helm-lab -f values-prod.yaml

You are directly changing the cluster from your terminal.

With Argo CD, you normally do not run helm upgrade manually. Instead:

Git repository contains chart + values
        ↓
Argo CD reads Git
        ↓
Argo CD renders Helm chart
        ↓
Argo CD applies Kubernetes manifests

So in Argo CD, Helm is mostly used as a template renderer, and Argo CD controls sync and drift correction.

Before Argo CD, answer this

In GitOps, where should the desired image tag usually be stored?

A. Only in your local terminal command with --set image.tag=...
B. In Git, for example inside values-prod.yaml
C. Only inside the live Kubernetes Deployment using kubectl edit
D. Only inside Helm history

After you answer, we will create the Argo CD Application manifest for this Helm chart.