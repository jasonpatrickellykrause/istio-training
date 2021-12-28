# Lab: Egress gateway

In this lab we'll show how to route traffic for external services through the egress gateway. We'll create the Istio resource to route all traffic to the Github API (e.g. api.github.com) through the egress gateway. We'll use a simple curl pod for testing. 

## Installing egress gateway

Egress gateway is installed automatically in Istio's demo profile. If you used any other profiles you can install the Egress gateway by updating the Istio operator:

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: demo-installation
spec:
  profile: demo
  components:
    egressGateways:
    - name: istio-egressgateway
      enabled: true
```

Save the above YAML to `iop-egress.yaml` and deploy it using `kubectl apply -f iop-egress.yaml`.

First we'll create ServiceEntry that allows access to an external service. We'll use Github as our external service, but this could be literally any other external URL.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: github
spec:
  hosts:
  - api.github.com
  ports:
  - number: 80
    name: http-port
    protocol: HTTP
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
```

Save the above YAML to `github-se.yaml` and deploy it using `kubectl apply -f github-se.yaml`.

Next, we'll create a Gateway resource and a destination rule configuring the egress gateway.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istio-egressgateway
spec:
  selector:
    istio: egressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - api.github.com
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: egressgateway-for-github
spec:
  host: istio-egressgateway.istio-system.svc.cluster.local
  subsets:
  - name: github
```

Save the above YAML to `egress.yaml` and deploy both resources using `kubectl apply -f egress.yaml`.

Finally, we can create a VirtualService to direct the traffic from the sidecars (matching on the `mesh` gateway) to the egress gateway, and from the egress gateway (match on`istio-egressgateway`) to the Github API:


```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: github-to-egress-gateway
spec:
  hosts:
  - api.github.com
  gateways:
  - istio-egressgateway
  - mesh
  http:
  - match:
    - gateways:
      - mesh
      port: 80
    route:
    - destination:
        host: istio-egressgateway.istio-system.svc.cluster.local
        subset: github
        port:
          number: 80
      weight: 100
  - match:
    - gateways:
      - istio-egressgateway
      port: 80
    route:
    - destination:
        host: api.github.com
        port:
          number: 80
      weight: 100
```
Save the above YAML to `vs.yaml` and deploy it `kubectl apply -f vs.yaml`.

We can now run a Pod inside the cluster and make a request to e.g. `https://api.github.com/users/peterj`.

Let's run the `curl` pod first:

```sh
kubectl run curl --image=radial/busyboxplus:curl -i --tty --rm
```

Once we have the terminal inside the container we can send a request to `https://api.github.com/peterj`:

```sh
curl https://api.github.com/users/peterj
{
  "login": "peterj",
  "id": 11080940,
  "node_id": "MDQ6VXNlcjExMDgwOTQw",
  "avatar_url": "https://avatars.githubusercontent.com/u/11080940?v=4",
  "gravatar_id": "",
  "url": "https://api.github.com/users/peterj",
  "html_url": "https://github.com/peterj",
...
```

The output is the same as if we'd run the curl command outside of the cluster. However, since we routed the traffic through the egress gateway we can check the logs from that pod. Type `exit` to exit the curl pod and then look at the logs from the egress gateway pod:

```sh
$ kubectl logs -l istio=egressgateway -n istio-system
...
-system resources:1 size:4.0kB resource:default
2021-12-27T23:55:28.244927Z     info    Readiness succeeded in 568.71563ms
2021-12-27T23:55:28.245713Z     info    Envoy proxy is ready
[2021-12-28T00:07:05.710Z] "GET /users/peterj HTTP/2" 301 - via_upstream - "-" 0 0 18 17 "10.92.1.18" "curl/7.35.0" "7cf874bc-0a2b-989e-8b1c-9a3c47d7ffab" "api.github.com" "192.30.255.116:80" outbound|80||api.github.com 10.92.2.21:55642 10.92.2.21:8080 10.92.1.18:53774 - -
[2021-12-28T00:07:08.595Z] "GET /users/peterj HTTP/2" 301 - via_upstream - "-" 0 0 8 8 "10.92.1.18" "curl/7.35.0" "4c739d80-7c40-97b3-a014-16861681a711" "api.github.com" "192.30.255.116:80" outbound|80||api.github.com 10.92.2.21:55642 10.92.2.21:8080 10.92.1.18:53774 - -
[2021-12-28T00:07:12.261Z] "GET /users/peterj HTTP/2" 301 - via_upstream - "-" 0 0 16 15 "10.92.1.18" "curl/7.35.0" "1050d36d-0bf0-9118-96c3-ce77f824b1a5" "api.github.com" "192.30.255.116:80" outbound|80||api.github.com 10.92.2.21:55670 10.92.2.21:8080 10.92.1.18:53816 - -
```

You'll notice all requests we've made to the external service were logged in the egress gateway.

# Cleanup

To delete everything, run:

```sh
kubectl delete se github
kubectl delete gw istio-egressgateway
kubectl delete dr egressgateway-for-github
kubectl delete vs github-to-egress-gateway
```