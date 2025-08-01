---
docType: "Chapter"
chapterTitle: "Exposing services through Istio Ingress Gateway"
title: "Exposing services through Istio Ingress Gateway"
description: "Meshery, collaborative Kubernetes manager"
lectures: 12
weight: 3
---

{{< chapterstyle >}}
The components deployed on the service mesh by default are not exposed outside the cluster. An Ingress Gateway is deployed as a Kubernetes service of type LoadBalancer (or NodePort). To make Bookinfo accessible external to the cluster, you have to create an `Istio Gateway` for the Bookinfo application and also define an `Istio VirtualService` with the routes we need.

<br />
<br />

<h2 class="chapter-sub-heading"> Inspecting the Istio Ingress Gateway</h2>

<br />
The ingress gateway gets exposed as a normal Kubernetes service of type LoadBalancer (or NodePort):

```sh
kubectl get svc istio-ingressgateway -n istio-system -o yaml
```

Because the Istio Ingress Gateway is an Envoy Proxy you can inspect it using the admin routes. First find the name of the istio-ingressgateway:

```sh
kubectl get pods -n istio-system
```

Copy and paste your ingress gateway's pod name. Execute:

```sh
kubectl -n istio-system exec -it <istio-ingressgateway-...> bash
```

You can view the statistics, listeners, routes, clusters and server info for the Envoy proxy by forwarding the local port:

```sh
curl localhost:15000/help
curl localhost:15000/stats
curl localhost:15000/listeners
curl localhost:15000/clusters
curl localhost:15000/server_info
```

See the [admin docs](https://www.envoyproxy.io/docs/envoy/latest/operations/admin) for more details.

Also it can be helpful to look at the log files of the Istio ingress controller to see what request is being routed.

Before we check the logs, let us get out of the container back on the host:

```sh
exit
```

Now let us find the ingress pod and output the log:

```sh
kubectl logs istio-ingressgateway-... -n istio-system
```

<h2 class="chapter-sub-heading">View Istio Ingress Gateway for Bookinfo</h2>
<br />

<h3 class="chapter-sub-heading"> View the Gateway and VirtualServices</h3>

Check the created `Istio Gateway` and `Istio VirtualService` to see the changes deployed:

```sh
kubectl get gateway
kubectl get gateway -o yaml

kubectl get virtualservices
kubectl get virtualservices -o yaml
```

<h3 class="chapter-sub-heading">
  Find the external port of the Istio Ingress Gateway by running:
</h3>

```sh
kubectl get service istio-ingressgateway -n istio-system -o wide
```

To just get the first port of istio-ingressgateway service, we can run this:

```sh
kubectl get service istio-ingressgateway -n istio-system --template='{{(index .spec.ports 1).nodePort}}'
```

<h3 class="chapter-sub-heading"> Create a DNS entry:</h3>

Modify you local `/etc/hosts` file to add an entry for your sample application.

`127.0.0.1. bookinfo.meshery.io`

The HTTP port is usually 31380.

Or run these commands to retrieve the full URL:

```sh
echo "http://$(kubectl get nodes --selector=kubernetes.io/role!=master -o jsonpath={.items[0].status.addresses[?\(@.type==\"InternalIP\"\)].address}):$(kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.spec.ports[1].nodePort}')/productpage"
```

Docker Desktop users please use `http://localhost/productpage` to access product page in your browser.

In case you are using a managed kubernetes cluster like AKS, EKS, or GCE please follow the procedure described below:

- Get the external IP of the service istio-ingressgateway using the following command:

  ```sh
  kubectl get service istio-ingressgateway -n istio-system
  ```

- Using Meshery, navigate to the Custom yaml page, and apply the manifest given below to allow all hosts instead of allowing bookinfo.meshery.io only and you are good to access the page using the following url `http://<external-ip of istio-ingressgateway>/productpage`.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: bookinfo
spec:
  gateways:
    - sample-app-gateway
  hosts:
    - "*"
  http:
    - match:
        - uri:
            exact: /productpage
        - uri:
            prefix: /static
        - uri:
            exact: /login
        - uri:
            exact: /logout
        - uri:
            prefix: /api/v1/products
      route:
        - destination:
            host: productpage
            port:
              number: 9080
```

<h2 class="chapter-sub-heading"> Apply default destination rules</h2>

Before we start playing with Istio's traffic management capabilities we need to define the available versions of the deployed services. They are called subsets, in destination rules.

Using Meshery, navigate to the Custom yaml page, and apply the below to create the subsets for BookInfo:

```sh
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: productpage
spec:
  host: productpage
  subsets:
    - name: v1
      labels:
        version: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
    - name: v3
      labels:
        version: v3
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: ratings
spec:
  host: ratings
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
    - name: v2-mysql
      labels:
        version: v2-mysql
    - name: v2-mysql-vm
      labels:
        version: v2-mysql-vm
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: details
spec:
  host: details
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

This creates destination rules for each of the BookInfo services and defines version subsets

In a few seconds we should be able to verify the destination rules created by using the command below:

```sh
kubectl get destinationrules


kubectl get destinationrules -o yaml
```

<h2 class="chapter-sub-heading"> Browse to BookInfo</h2>

Browse to the website of the Bookinfo. To view the product page, you will have to append
`/productpage` to the url.

<h3 class="chapter-sub-heading"> Reload Page</h3>

Now, reload the page multiple times and notice how it round robins between v1, v2 and v3 of the reviews service.

<h2 class="chapter-sub-heading">Inspect the Istio proxy of the productpage pod</h2>

To better understand the istio proxy, let's inspect the details. Let us `exec` into the productpage pod to find the proxy details. To do so we need to first find the full pod name and then `exec` into the istio-proxy container:

```sh
kubectl get pods
kubectl exec -it productpage-v1-... -c istio-proxy  sh
```

Once in the container look at some of the envoy proxy details by inspecting it's config file:

```sh
ps aux
ls -l /etc/istio/proxy
cat /etc/istio/proxy/envoy-rev0.json
```

For more details on envoy proxy please check out their [admin docs](https://www.envoyproxy.io/docs/envoy/v1.5.0/operations/admin).

As a last step, lets exit the container:

```sh
exit
```

<br />
<h3>Alternative: Manual installation</h3>
Follow this if the above steps did not work for you

<br />
<br />

<h4 class="chapter-alt-heading"> Default destination rules</h4>

Run the following command to create default destination rules for the Bookinfo services:

```sh
kubectl apply -f samples/bookinfo/networking/destination-rule-all-mtls.yaml
```

<h4 class="chapter-alt-heading">
  Configure the Bookinfo route with the Istio Ingress gateway
</h4>

We can create a virtualservice & gateway for bookinfo app in the ingress gateway by running the following:

```sh
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
```

{{< /chapterstyle >}}