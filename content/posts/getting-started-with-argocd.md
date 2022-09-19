---
title: "Getting started with Argo CD - Declarative GitOps for Kubernetes"
date: 2022-09-18
draft: false
tags:
 - "kubernetes"
 - "argocd"
 - "tech"
categories:
 - "techstuff"
---

This blog post with give you a quick and brief run through of standing up Argo CD and using it to deploy a small application. Argo CD is a K8s controller, responsible for continuously monitoring all running applications and comparing their live state to the desired state specified in the Git repository. It identifies deployed 
applications with a live state that deviates from the desired state as OutOfSync.

Argo allows engineering teams to deploy and manage applications without having to learn a lot about Kubernetes, and without needing full access to the Kubernetes system, which is great when you start to learn just how overwhelming all this kubernetes stuff really is!

Basically? It polls git and ensures your deployment matches the source on a periodic basis enabling continuous delivery! Awesome!

## Prerequisites
* A Kubernetes Cluster ( Can be either On-Prem, AKS, EKS, GKE, Kind ) - I'm using minikube!
* Helm, kubectl installed.

## Getting Started

Create the namespace to house all the argo components. 
`kubectl create namespace argocd`

Add the helm repo for Argo
`helm repo add argo https://argoproj.github.io/argo-helm`

Install the helm chart for argocd this will setup a number of components in the argocd namespace.
`helm install argocd -n argocd argo/argo-cd`

Check the pods are available:
```sh
$ kubectl get pods -n argocd                                      
NAME                                               READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                    1/1     Running   0          71s
argocd-applicationset-controller-558cd44b5-r4jgq   1/1     Running   0          71s
argocd-dex-server-77699b54cb-56bkv                 1/1     Running   0          71s
argocd-notifications-controller-6cf995fd6-ztbx9    1/1     Running   0          71s
argocd-redis-76966cd759-dhkl7                      1/1     Running   0          71s
argocd-repo-server-6d7867f4c7-24cws                1/1     Running   0          71s
argocd-server-fd5c9c7db-gd694                      1/1     Running   0          71s
```

Retrieve the admin account password:
`kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`

Forward the ports:
`kubectl port-forward service/argocd-server -n argocd 8080:443`

Access localhost:8080 in your browser and login using username admin and the password that you retrieved earlier.

![](https://i.imgur.com/MiV5YF8.png)

Before this next step, initialise a empty repository in your source control of choice and create the folder `sample-nginx` with the file `manifest.yml` inside. I went with Github and my example is here: https://github.com/lukayeh/getting-started-with-argo/blob/main/sample-nginx/manifest.yml

![](https://i.imgur.com/v5bp2As.png)

In the webUI create a new application using this YAML as a template 
```yml
project: default
source:
  repoURL: 'https://github.com/lukayeh/getting-started-with-argo.git'
  path: sample-nginx/
  targetRevision: HEAD
  directory:
    jsonnet:
      tlas:
        - name: ''
          value: ''
destination:
  server: 'https://kubernetes.default.svc'
  namespace: default
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

The repoURL should point to your publicly accessible GIT Repostiory hosting your kuberenetes definitions mine contains the following:
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bgd-deployment
  labels:
    app: bgd
spec:
  replicas: 4
  selector:
    matchLabels:
      app: bgd
  template:
    metadata:
      labels:
        app: bgd
    spec:
      containers:
      - image: quay.io/redhatworkshops/bgd:latest
        name: bgd
        env:
        - name: COLOR
          value: "blue"
        resources: {}
---
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: bgd
  name: bgd
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: bgd
---
```

Once deployed you should start to see the pods and the service standup.

The pods:
```sh
kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
bgd-deployment-68c5598bcb-2gqbm   1/1     Running   0          72s
bgd-deployment-68c5598bcb-45ltl   1/1     Running   0          66s
bgd-deployment-68c5598bcb-57zsq   1/1     Running   0          72s
bgd-deployment-68c5598bcb-dc98m   1/1     Running   0          65s
```

The service:

```sh
kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
bgd          ClusterIP   10.107.120.118   <none>        8080/TCP   11m
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP    150m
```

Now forward the port 8080 locally from the service and access it, along with other information you should see a blue square!

`kubectl port-forward service/bgd 8088:8080`

(I Forwarded on port 8088 as I already had something running on 8080!)

![](https://i.imgur.com/Lb2cteB.png)

Now back to your `manifest.yml` update the env section changing the color of the square:
```yaml
        env:
        - name: COLOR
          value: "green"
        resources: {}
```

and commit this to the main branch. Now wait for ArgoCD to pick up this change (default sync time is 3 minutes).

Once deployed you should see that the last sync result is "Sync OK" and that the pods have been updated.

Once again visit your localhost:8088 you should see that the square has changed color :shock:

![](https://i.imgur.com/zuaMvEI.png)

## Conclusion

So in conclusion this post has quickly demonstrated how to get Argo CD setup and deploying from a git repository in future I may go into further details about how you can use Helm. 

As you can tell Argo is extremely simple to setup and can be as complex as you'd like going from a simple application like the one we've deployed today to a huge complex kubernetes definition the worlds your oyster!
