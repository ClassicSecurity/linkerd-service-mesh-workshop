# Lab 3 - Ingressing and Egressing with Linkerd

<!-- Services running on the Linkerd service mesh by default are not exposed outside the cluster. -->

Linkerd's control plane does not include ingress or egress gateways. Linkerd allows you choice of your preferred ingress (and egress) controller.

## How to use Ingress with Linkerd

In case you're anticipating infusing Linkerd into your ingress controller's pods there is some setup required. Linkerd discovers services dependent on the `:authority` or `Host` header. This permits Linkerd to comprehend what service a request is bound for without being subject to DNS or IPs.

In this workshop, you will use the NGINX Ingress Controller with Linkerd.

## 3.1 Installing NGINX Ingress Controller

Using Meshery, select the Linkerd from the `Management` menu, and:

1. Click the (+) icon on the `Apply Service Mesh Configuration` card and select `NGINX Ingress Controller` to install the latest version of KIC.

## 3.2 Setting up ingress controller with the sample application deployed

Using Meshery, click the (▶️) icon on the `Apply Custom Configuration` card and apply the following manifest to your cluster:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: web-ingress
  namespace: emojivoto
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_set_header l5d-dst-override $service_name.$namespace.svc.cluster.local:$service_port;
      grpc_set_header l5d-dst-override $service_name.$namespace.svc.cluster.local:$service_port;

spec:
  rules:
    - host: example.com
      http:
        paths:
          - backend:
              serviceName: web-svc
              servicePort: 80
```

Note these lines:

```sh
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_set_header l5d-dst-override $service_name.$namespace.svc.cluster.local:$service_port;
      grpc_set_header l5d-dst-override $service_name.$namespace.svc.cluster.local:$service_port;
```

This model joins the two mandates that NGINX utilizes for proxying HTTP and gRPC traffic. Practically speaking, it is just important to set either the proxy_set_header or grpc_set_header mandate, contingent upon the protocol utilized by the administration, anyway NGINX will overlook any orders that it needn't bother with.

Nginx will include a l5d-dst-abrogate header to train Linkerd what administration the solicitation is bound for. You'll need to incorporate both the Kubernetes administration FQDN (web-svc.emojivoto.svc.cluster.local) and the objective servicePort.

To test this, you need to get the external IP of your controller which you can get by running

```sh
kubectl get svc --all-namespaces \
  -l app=nginx-ingress,component=controller \
  -o=custom-columns=EXTERNAL-IP:.status.loadBalancer.ingress[0].ip
```

You can now curl to your service without using port-forward

```sh
curl -H "Host: example.com" http://{external-ip}
```

<img src="../img/go.svg" width="32" height="32" align="left"
style="padding-right:4px;" />

## [Continue to Lab 4](../lab-4/README.md) - Exploring Linkerd Dashboard

<br />
<hr />

Alternative, manual installation steps are provided for reference below. No need to execute these if you have performed the steps above.

<hr />

## 3.1 Installing NGINX Ingress Controller

- Install ingress controller using Docker Desktop

```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.40.2/deploy/static/provider/cloud/deploy.yaml
```

- Install the ingress controller using Minikube

```sh
minikube addons enable ingress
```
