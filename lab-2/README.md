# Lab 2 - Deploy a Sample Application

To play with Linkerd and demonstrate some of it's capabilities, you will deploy the sample application, EmojiVoto.

## The EmojiVoto sample application

Emojivoto is a sample microservice application that allows users to vote for their favorite emoji. It displays votes received on a leaderboard. May the best emoji win!

![Emojivoto's architecture](https://raw.githubusercontent.com/BuoyantIO/emojivoto/main/assets/emojivoto-topology.png)
_Emojivoto's architecture. Source: Bouyant_

It’s worth noting that these services have no dependencies on Linkerd, but make an interesting service mesh example, particularly because of the variety of services, languages and versions for the reviews service.

### <a name="auto"></a> A note on sidecar proxy injection

The Linkerd sidecar proxy can be either manually or automatically injected into your application's pods.

A sidecar injector is used for automating the injection of the Linkerd proxy into your application's pod spec. The Kubernetes admission controller enforces this behavior send sending a webhook request the the sidecar injector every time a pod is to be scheduled. This injector inspects resources for a Linkerd-specific annotation (linkerd.io/inject: enabled). When that annotation exists, the injector mutates the pod's specification and adds both an init container as well as a sidecar containing the proxy itself.

As part of Linkerd deployment in [Lab 1](../lab-1/README.md), you have deployed the sidecar proxy injector, which should be running in your control plane.To verify, execute this command:

```sh
kubectl get deployment linkerd-proxy-injector -n linkerd
```

Expected Output:

```sh
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
linkerd-proxy-injector   1/1     1            1           9m49s
```

Examine the annotation added to the `linkerd` namespace. Execute this command:

```sh
kubectl get namespace linkerd -o yaml
```

Expected Output:

```sh
apiVersion: v1
kind: Namespace
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Namespace","metadata":{"annotations":{"linkerd.io/inject":"disabled"},"labels":{"config.linkerd.io/admission-webhooks":"disabled","linkerd.io/control-plane-ns":"linkerd","linkerd.io/is-control-plane":"true"},"name":"linkerd"}}
    linkerd.io/inject: disabled
  creationTimestamp: "2020-10-09T18:21:48Z"
  labels:
    config.linkerd.io/admission-webhooks: disabled
    linkerd.io/control-plane-ns: linkerd
    linkerd.io/is-control-plane: "true"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:kubectl.kubernetes.io/last-applied-configuration: {}
          f:linkerd.io/inject: {}
        f:labels:
          .: {}
          f:config.linkerd.io/admission-webhooks: {}
          f:linkerd.io/control-plane-ns: {}
          f:linkerd.io/is-control-plane: {}
      f:status:
      ...
```

## Deploying the sample application

To deploy the Emojivoto application, follow these steps:

1. Using Meshery, navigate to the Linkerd management page.
1. Enter `default` in the `Namespace` field.
1. Click the (+) icon on the `Sample Application` card and select `Emojivoto Application` from the list.

This will do three things:

1. Deploy `Emojivoto Application` in Emojivoto namespace.
1. Deploys all the Emojivoto services and replica's in the required namespace
1. Injects Linkerd into each component of the `Emojivoto Application`

<small>[Steps for manual injection](#appendix)</small>

### <a name="verify"></a> Verify Emojivoto deployment

1. Verify that previous deployments are all in a state of AVAILABLE before continuing. **Do not proceed until they are up and running.**

   ```sh
   watch -n emojivoto kubectl get deployment
   ```

2. Inspect the details of the pods

   Let us look at the details of the pods:

   ```sh
   watch -n emojivoto kubectl get po
   ```

   Let us look at the details of the services:

   ```sh
   watch -n emojivoto kubectl get svc
   ```

   Choose one of Emojivoto's services (e.g. `web-svc`), and view it's sidecar configuration:

   ```sh
   kubectl -n emojivoto get svc

   kubectl -n emojivoto describe service svc/web-svc
   ```

Let's look at the application deployment by port-forwarding the `web-svc` service:

```sh
kubectl -n emojivoto port-forward svc/web-svc 8080:80
```

#### <a name="linkerd_inject"></a> Inject Linkerd into the sample application

The emojivoto application is a standalone Kubernetes application that uses a mix of gRPC and HTTP calls to allow the users to vote on their favorite emojis, which means the application can run standalone without support from Linkerd service mesh. Injecting linkerd into our sample application

```sh
kubectl -n emojivoto patch -f https://run.linkerd.io/emojivoto.yml -p '
spec:
  template:
    metadata:
      annotations:
        linkerd.io/inject: enabled
'
```

This command retrieves all of the deployments running in the emojivoto namespace, runs the manifest through linkerd inject, and then reapplies it to the cluster. The linkerd inject command adds annotations to the pod spec instructing Linkerd to add (“inject”) the proxy as a container to the pod spec.

You've now added Linkerd to existing services! Just as with the control plane, it is possible to verify that everything worked the way it should with the data plane. To do this check, run:

```sh
linkerd -n emojivoto check --proxy
```

Linkerd, in contrast to istio annotates the resources (namespaces, deployment workloads) rather than labelling them.

<img src="../img/go.svg" width="32" height="32" align="left"
style="padding-right:8px;" />

## [Continue to Lab 3](../lab-3/README.md) - Using Ingress with Linkerd

<br />
<hr />
Alternative, manual installation steps below. No need to execute, if you have performed the steps above.
<hr />

## <a name="appendix"></a> Appendix - Alternative Manual Steps

### Deploy emojivoto application

Install emojivoto into the emojivoto namespace by running:

```sh
curl -sL https://run.linkerd.io/emojivoto.yml \
  | kubectl apply -f -
```

Before we mesh it, let's take a look at the app. If you're using Docker Desktop at this point you can visit http://localhost directly. If you're not using Docker Desktop, we'll need to forward the web-svc service. To forward web-svc locally to port 8080, you can run:

```sh
kubectl -n emojivoto port-forward svc/web-svc 8080:80
```

### Deploy Emojivoto

Applying this yaml file included in the Linkerd package you collected in https://run.linkerd.io/emojivoto.yml will deploy the sample app into your cluster.

```sh
kubectl apply -f https://run.linkerd.io/emojivoto.yml
```

#### <a name="linkerd_inject"></a> Inject Linkerd into the sample application

The emojivoto application is a standalone Kubernetes application that uses a mix of gRPC and HTTP calls to allow the users to vote on their favorite emojis, which means the application can run standalone without support from linkerd service mesh.
Now we will be injecting linkerd into our sample application

```sh
kubectl get -n emojivoto deploy -o yaml \
 | linkerd inject - \
 | kubectl apply -f -
```

This command retrieves all of the deployments running in the emojivoto namespace, runs the manifest through linkerd inject, and then reapplies it to the cluster. The linkerd inject command adds annotations to the pod spec instructing Linkerd to add (“inject”) the proxy as a container to the pod spec.

You've now added Linkerd to existing services! Just as with the control plane, it is possible to verify that everything worked the way it should with the data plane. To do this check, run:

```sh
linkerd -n emojivoto check --proxy
```
