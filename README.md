
# Kustomize for Hello World Python

Create the base folder containing all the manifest that needed by both environment. 

```shell
$ cd hello-argocd
$ tree base
base
|-- deployment.yaml
|-- kustomization.yaml
|-- route.yaml
|-- service.yaml
`-- serviceaccount.yaml
```

The `kustomization.yaml` is the indication that your apps can use kustomize to generate the manifests. The following shows that you want to use all the manifest listed in the array.

```shell
$ cat base/kustomization.yaml
resources:
  - deployment.yaml
  - serviceaccount.yaml
  - service.yaml
  - route.yaml
```

## Creating kustomization for different environment

Kustomize works by having overlays for differrent environment, you can create `overlays` folder and each separate folder for each environment
```shell
$ tree overlays
overlays
|-- dev
|   |-- kustomization.yaml
|   `-- route.yaml
`-- prod
    |-- kustomization.yaml
    `-- route.yaml
```    

Let's now look at the `dev` environment, the following shows that we want to apply `namespace: argocd-dev`, we also want to include whatever we have in the `base`. 

```shell
namespace: argocd-deve
resources:
  - ../../base/
patches:
  - path: route.yaml
```

The `route.yaml` needed some customization hence we create a seperate route.yaml that has different properties compare to the base.
```shell
$ cat overlays/dev/route.yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: hello-world
  labels:
    app: hello-world
spec:
  # Remember to change it for different cluster domain
  host: hello-world-argocd-dev.apps.cluster-dqk9j.dqk9j.sandbox963.opentlc.com # the URL has `argocd-dev` suffix
  port:
    targetPort: http
  tls:
    termination: edge
  to:
    kind: Service
    name: hello-world
    weight: 100
  wildcardPolicy: None 
```

## How it works

The following method, doesn't require connection to the cluster.
```shell
$ cd hello-argocd
$ oc kustomize base # validates your yaml resources
$ oc kustomize overlays/prod

Another method
```shell
$ oc apply -k base --dry-run -o yaml # the `k` option to indicate the oc/kubectl to use kustomize method
$ oc apply -k overlays/prod --dry-run -o yaml
```

Once ready, we can omit the `--dry-run` option and applied it directly to the cluster
```shell
$ oc whoami
$ oc project
Using project "argocd-dev" on server "https://api.cluster-dqk9j.dqk9j.sandbox963.opentlc.com:6443".
$ oc apply -k overlays/dev
NAME                               READY   STATUS    RESTARTS   AGE
pod/hello-world-7675bbc66d-lp9tt   1/1     Running   0          4s

NAME                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/hello-world   ClusterIP   172.30.205.231   <none>        8080/TCP   4s

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello-world   1/1     1            1           4s

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/hello-world-7675bbc66d   1         1         1       4s

NAME                                   HOST/PORT                                                                PATH   SERVICES      PORT   TERMINATION   WILDCARD
route.route.openshift.io/hello-world   hello-world-argocd-dev.apps.cluster-dqk9j.dqk9j.sandbox963.opentlc.com          hello-world   http   edge          None

$ curl -k https://hello-world-argocd-dev.apps.cluster-dqk9j.dqk9j.sandbox963.opentlc.com
Hello world - v1.0
```

Clean up
```shell
$ oc delete all -l app=hello-world
```