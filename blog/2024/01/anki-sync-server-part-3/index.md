---
layout: blog
title: K8S The Number of the Beast Part 3
date: 2024-01-22T16:37:15-06:00
lastMod: 2024-01-22T16:37:15-06:00
categories: self-hosted
tags:
  - docker
  - anki
  - containers
  - kubernetes
  - self-hosted
  - github
  - argocd
  - traefik
  - longhorn
description: The last part of a 3 part write up deploying an Anki Sync Server
disableComments: true
draft: false
---
## Deploying in Kubernetes
Now that we have the container image I can work on deploying this to my Kubernetes Cluster. Now obviously I can deploy this to Docker and just be done with it, but a good chunk of my self hosted applications are deployed to Kubernetes. I don't have any greater reasons for deploying in Kubernetes other than it's beneficial for my understanding and helps with my day job. Now with deployments in Kubernetes I don't have to worry so much if/when I take down a node for upgrade as the container will just move to another node for high up time.

## Breaking it Down
Compared to deploying with `docker-compose` yaml, there's quite a bit more needed to deploy in Kubernetes. Containers deployed in Kubernetes are grouped as Pods, which can contain 1 or more containers, but for our use case we will only have 1 container in our pod. What we need to decide on is if our Application is stateful or not, which will help to determine if we should use a Deployment or a Statefulset. For my use case I only need 1 container and the application itself isn't so state sensitive that I need to use a Statefulset. Next is Storage, we do need a big amount of storage but we need it to persist if the container is terminated, within my Kubernetes Cluster I'm using [Longhorn](https://longhorn.io/) for my Storage Class, this offers a number of useful features such as replicas and NFS/S3 backup solutions. Finally I'll need a Reverse Proxy/Ingress solution to allow traffic from outside the Kubernetes Cluster to route to my Application, for that I'll be using [Traefik](https://traefik.io/).

So breaking down the yaml files we'll need are the following...
- `deployment.yaml` (defines our application)
- `ingressroute.yaml` (defines our ingress routing)
- `pvc.yaml` (defines our persistent volume claim)
- `service.yaml` (defines our service/routing)
- `secrets.yaml` (defines our environment variables as a Kubernetes Secret)

### deployment.yaml
The following is an extremely basic Kubernetes deployment yaml that will deploy 1 replica (e.g. 1 pod) that is our `anki-syncserver`. The pod exposes port `8080` just like our `docker-compose` example did.
```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: anki-syncserver
  labels:
    app: anki-syncserver
spec:
  replicas: 1
  selector:
    matchLabels:
      app: anki-syncserver
  template:
    metadata:
      labels:
        app: anki-syncserver
    spec:
      containers:
        - name: anki-syncserver
          image: ghcr.io/sinicide/anki-syncserver:v1.0.2
          envFrom:
          - secretRef:
              name: anki-syncserver-secrets
          ports:
            - name: http
              containerPort: 8080
          volumeMounts:
          - name: anki-syncserver-pv
            mountPath: "/data"
      volumes:
        - name: anki-syncserver-pv
          persistentVolumeClaim:
            claimName: anki-syncserver-claim
```

The `volumes` section here defines the referenced Persistent Volume Claim, which our Pod will use.
```yaml
volumes:
	- name: anki-syncserver-pv
	  persistentVolumeClaim:
		claimName: anki-syncserver-claim
```

Then we mount that Claim via the `volumeMounts` sections of the Container definition, and in my case we're mounting the PVC to `/data`
```yaml
volumeMounts:
  - name: anki-syncserver-pv
	mountPath: "/data"
```

Finally we're taking the key/value pairs from our Kubernetes Secret and using them as Environment Variables within the Container.
```yaml
envFrom:
  - secretRef:
	  name: anki-syncserver-secrets
```

### pvc.yaml
The persistent volume claim is fairly straight forward, it simply defines which Storage Class is used and the requested resource values.
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: anki-syncserver-claim
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: longhorn-retain
  resources:
    requests:
      storage: 512M
```

Since I'm not entirely sure how much disk space I'll truly need to sync, I figured `512M` will be a good starting point. The Storage Class itself will allow for volume expansion and my custom Storage Class also enables the persistent volume to be retained when the pod is terminated. I'm also not overtly concerned with backing up this data volume as I don't consider this data necessarily mission critical, but Longhorn create multiple Replicas of my data and does allow for backing up to an NFS share or an S3 bucket as I mentioned earlier.

### service.yaml
The service resource defines an endpoint and can be of type Load Balancer for requests outside the cluster to route to our application. However we'll be using Traefik for our Ingress routing instead, meaning we just define a basic service resource, Traefik will have it's own Load Balancer Service defined which will route traffic on port `80` to our application's port `8080`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: anki-syncserver
spec:
  selector:
    app: anki-syncserver
  ports:
    - protocol: TCP
      port: 8080
      name: http
```

### inrgressroute.yaml
Since I'm using Traefik for Ingress, we need to define a Custom Resource called `IngressRoute` that will match a rule to route the DNS hostname `anki-temp.sin.lan` on port 80 to our service on port `8080`. Traefik has this concept of `entryPoints` which are typically just `http/80` or `https/443` which we can then redirect to our application's specific port. Since there's no SSL configuration for Anki Sync Server out of the box and it's not a huge deal for me to use a super crackable password, I don't have a need to offload SSL handling with my Ingress Controller. I've simply opted to just use the http endpoint called `web` for the time being. I primarily still use Self-Signed Certificates in my home network and these typically will fail validation in most application unless there's a way to specify a Truststore, so I don't want to bother with that hassle for now.
```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: anki-syncserver-http
spec:
  entryPoints:
    - web
  routes:
  - match: Host(`anki-temp.sin.lan`)
    kind: Rule
    services:
      - name: anki-syncserver
        port: 8080
```

With this, I can reach my Anki Sync Server at the following address `http://anki-temp.sin.lan/`

### secrets.yaml
Since I need to provide an Environment Variable with our username+password, I've opted to use a Kubernetes Secret to hold this sensitive information as opposed to simply defining it within the Deployment yaml. So in this resource, I'm providing a key/value data, our Environment Variable `SYNC_USER1` and it's associating value which is composed of the username and password separated by a colon `anki_username:anki_password`
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: anki-syncserver-secrets
type: Opaque
stringData:
  SYNC_USER1: "anki_username:anki_password"
```

## The CI/CD Way
I've been playing around with around with ArgoCD for my Continuous Deployment solution in my Kubernetes cluster, so of course I had to get my Anki Sync Server Deployment going here. This allows me to update my github repo and ArgoCD will pull down the changes when detected and deploy the changes in my Kubernetes Cluster.

ArgoCD provides a Custom Resource Definition for deploying Applications in Kubernetes, This yaml can reference a github repo or Helm chart that it'll use to deploy and keep watch on. So the following is what I'm using, which is extremely bare minimum to get my Anki Sync Server deployed in my Kubernetes Cluster.
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: anki-syncserver
  namespace: argocd
  labels:
    name: anki-syncserver
  # remove this for prod
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/sinicide/sh-repo
    targetRevision: HEAD
    path: yamls/anki-syncserver
    
  destination:
    server: https://kubernetes.default.svc
    namespace: apps

  syncPolicy:
    automated:
      prune: false
    syncOptions:
    - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  revisionHistoryLimit: 3
```

Please note, my [sh-repo](https://github.com/sinicide/sh-repo) for Anki Sync Server doesn't provide an example for the Kubernetes Secret yaml, so that's something that needs to be deployed manually before hand and needs to exist before ArgoCD can successfully deploy the Anki Sync Server Git Repo if you are using my github repo for your own deployment.

- [sh-repo](https://github.com/sinicide/sh-repo)
- [ArgoCD Yaml](https://github.com/sinicide/sh-repo/blob/main/argocd/application/anki-syncserver.yaml)
- [Anki-Syncserver](https://github.com/sinicide/sh-repo/tree/main/yamls/anki-syncserver)

## Conclusion
With my Anki Sync Server now deployed, I can point my local Anki Instances to the Sync Server `http://anki-temp.sin.lan` and will be able to sync progress and decks between my Desktop and Phone. Frankly it's working really well for all the extra work I went through to do this. Now comes the hard part which is learning Japanese with this Anki System, until next time.

じゃあね