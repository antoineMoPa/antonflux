# Antoine learns flux

## Context

I created a tiny deployment, called anton-deployment.

It contains the k8s dashboard and a landing page served by nginx.

Now I want to see how flux works.

## What the fudge is flux?

Let’s start using it and find out. (scroll to the end to find out)

## Bootstrapping flux

What is bootstrapping in flux?

If I understand correctly, it’s the process of starting from a k8s cluster and enabling flux in it.

First I created an empty private repo at named `antonflux`.

Then, I ran the bootstrapping command:

```bash
flux bootstrap git --url=ssh://git@github.com/antoineMoPa/antonflux.git --private-key-file= # put your private key path here
```

This made some files appear in my flux repo:

- https://github.com/antoineMoPa/antonflux/tree/789fcb092709849b2032c7dbaa480654bc1a4428
- https://github.com/antoineMoPa/antonflux/tree/376b7984497ebe2217673eb87ea3fb3dc8bc5a16

## Installing Podinfo in flux


Next, after cloning the repo locally, I’m following the instructions here to install podinfo:

https://fluxcd.io/flux/get-started/#add-podinfo-repository-to-flux

Why? I don't know.

```bash
# in the local antonflux repo:
mkdir clusters
mkdir clusters/anton-deployment

flux create source git podinfo \
   --url=https://github.com/stefanprodan/podinfo \
   --branch=master \
   --interval=1m \
   --export > ./clusters/anton-deployment/podinfo-source.yaml
```

I committed and pushed that new file.

I then got the current state of k8s and I could see the latest commit

```bash
$ flux get kustomizations --watch
NAME       	REVISION          	SUSPENDED	READY	MESSAGE
flux-system	main@sha1:5ccc71b5	False    	True 	Applied revision: main@sha1:5ccc71b5
```

However, a quick `kubectl get pods` shows that nothing changed.

Let’s continue:

```bash
flux create kustomization podinfo \
  --target-namespace=default \
  --source=podinfo \
  --path="./kustomize" \
  --prune=true \
  --wait=true \
  --interval=30m \
  --retry-interval=2m \
  --health-check-timeout=3m \
  --export > ./clusters/anton-deployment/podinfo-kustomization.yaml
```

Then I committed and pushed this new file.

The watch command gave more interesting output this time:

```bash
$ flux get kustomizations --watch
NAME       	REVISION          	SUSPENDED	READY	MESSAGE
flux-system	main@sha1:5ccc71b5	False    	True 	Applied revision: main@sha1:5ccc71b5
flux-system	main@sha1:5ccc71b5	False	Unknown	Reconciliation in progress
flux-system	main@sha1:5ccc71b5	False	Unknown	Reconciliation in progress
flux-system	main@sha1:5ccc71b5	False	Unknown	Reconciliation in progress
flux-system	main@sha1:5ccc71b5	False	Unknown	Reconciliation in progress
podinfo		False	False	waiting to be reconciled
podinfo		False	False	waiting to be reconciled
flux-system	main@sha1:5ccc71b5	False	True	Applied revision: main@sha1:b6a27a18
flux-system	main@sha1:b6a27a18	False	True	Applied revision: main@sha1:b6a27a18
podinfo		False	Unknown	Reconciliation in progress
podinfo		False	Unknown	Reconciliation in progress
podinfo		False	Unknown	Reconciliation in progress
podinfo		False	Unknown	Reconciliation in progress
podinfo		False	Unknown	Reconciliation in progress
podinfo		False	Unknown	Reconciliation in progress
```

And I see a new pod:

```bash
$ kubectl get po
NAME                               READY   STATUS    RESTARTS   AGE
anton-deployment-fb66cf6d8-pvwg8   1/1     Running   0          2d3h
anton-deployment-fb66cf6d8-x6xjd   1/1     Running   0          2d3h
podinfo-85c45f85db-9bplj           1/1     Running   0          13s
```

And new services:

```bash
$ kubectl -n default get deployments,services
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/anton-deployment   2/2     2            2           2d8h
deployment.apps/podinfo            2/2     2            2           7m48s

NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
service/anton-deployment   ClusterIP   10.101.165.243   <none>        80/TCP              3d6h
service/anton-service      NodePort    10.109.11.40     <none>        80:30007/TCP        3d
service/kubernetes         ClusterIP   10.96.0.1        <none>        443/TCP             3d6h
service/podinfo            ClusterIP   10.100.110.37    <none>        9898/TCP,9999/TCP   7m48s
```

Lets port-forward this service to see it:

```bash
kubectl port-forward service/podinfo 9898:9898 -n default
```

Nice, I can see this web ui:

![image](https://github.com/antoineMoPa/antonflux/assets/2675724/bef24fb5-28fc-4478-bcfd-f6b22a26ce4c)

It looks fancy, but I have no idea what to do with podinfo. Apparently it provides some APIs. I'll explore that... eventually.

## Migrating from my k8s hacky deployment to flux

Now that I have podinfo and have no idea what to do with it, let's do what I wanted originally and start
moving my old k8s manual deployment to flux.

First I moved some of my files to the new flux repo and added a kustomization file.

It's important to list the file in kustomization.yaml for anything to work.

```bash
$ tree
.
├── base
│   ├── admin-binding.yaml
│   ├── admin.yaml
│   ├── anton-deployment.yaml
│   ├── kustomization.yaml
│   ├── nginx-service.yaml
│   └── secrets.yaml
├── clusters
│   └── anton-deployment
│       ├── podinfo-kustomization.yaml
│       └── podinfo-source.yaml
└── flux-system
    ├── gotk-components.yaml
    ├── gotk-sync.yaml
    └── kustomization.yaml

5 directories, 11 files
$ cat base/anton-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: anton-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: nginx
        ## this part is a bit tricky. hiding for now.
        ## volumeMounts:
        ##   - name: static-content
        ##     mountPath: /usr/share/nginx/html/
      ## this part is a bit tricky. hiding for now.
      ## volumes:
      ##   - name: static-content
      ##     hostPath:
      ##       path: $STATIC_CONTENT_PATH
      ##       type: Directory
$ cat base/kustomization.yaml
resources:
  - admin-binding.yaml
  - admin.yaml
  - anton-deployment.yaml
  - nginx-service.yaml
  - secrets.yaml
```

Note that it's not a perfect replica of my previous setup, since I haven't found a nice way
to point to a relative path for the static html.

To check that everything is valid, let’s run it in dry mode:

```bash
kustomize build base/ | kubectl apply --dry-run=client -f -
```

I made sure to delete the deployment from my old k8s repo:

```bash
kubectl delete deployment anton-deployment
```

Then I committed and push the new changes in my flux repo.

Then… oops!

```bash
$ flux get kustomizations --watch
NAME       	REVISION          	SUSPENDED	READY	MESSAGE
flux-system	main@sha1:b6a27a18	False    	True 	Applied revision: main@sha1:b6a27a18
podinfo	master@sha1:dc830d02	False	True	Applied revision: master@sha1:dc830d02

flux-system	main@sha1:b6a27a18	False	Unknown	Reconciliation in progress
flux-system	main@sha1:b6a27a18	False	Unknown	Reconciliation in progress
flux-system	main@sha1:b6a27a18	False	Unknown	Reconciliation in progress
flux-system	main@sha1:b6a27a18	False	Unknown	Reconciliation in progress
**flux-system	main@sha1:b6a27a18	False	False	Service/anton-service namespace not specified: the server could not find the requested resource**
```

That’s not good:

```bash
Service/anton-service namespace not specified: the server could not find the requested resource
```

I added a namespace in the service:

```bash
metadata:
  name: anton-service
  namespace: antonflux
```

To actually create the namespace, I also needed to add a namespace.yaml file in base and link it in the komposition.

```bash
apiVersion: v1
kind: Namespace
metadata:
  name: antonflux
```

Then I had a port issue because my old deployment was already using the port 30007. I switched the port in the new version to 30008.

After pushing this change, my new flux cluster started working as expected:

![image](https://github.com/antoineMoPa/antonflux/assets/2675724/c083226b-4ca1-4768-b5d8-b28de513e238)


**Then end for now**

## What I learned

- Flux basically synchronizes kustomizations from a git repo to a k8s cluster
