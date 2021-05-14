# EnvoyFilter

Let's deploy the web frontend workload first using `kubectl apply -f envoy-demo-apps.yaml`.

We will create an Envoy filter that adds a header `api-version` to the HTTP response:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: api-header-filter
  namespace: default
spec:
  workloadSelector:
    labels:
      app: web-frontend
  configPatches:
  - applyTo: HTTP_FILTER
    match:
      context: SIDECAR_INBOUND
      listener:
        portNumber: 8080
        filterChain:
          filter:
            name: "envoy.http_connection_manager"
            subFilter:
              name: "envoy.router"
    patch:
      operation: INSERT_BEFORE
      value:
        name: envoy.lua
        typed_config:
          "@type": "type.googleapis.com/envoy.extensions.filters.http.lua.v3.Lua"
          inlineCode: |
            function envoy_on_response(response_handle)
              response_handle:headers():add("api-version", "v1")
            end
```

Save the above file to `envoy-header-filter.yaml` and deploy it using `kubectl apply -f envoy-header-filter.yaml`.

To see the header added, you can send a request to the Ingress gateway IP address:

```sh
$ export GATEWAY_URL=$(kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

$ curl -s -I -X HEAD  http://$GATEWAY_URL
HTTP/1.1 200 OK
x-powered-by: Express
content-type: text/html; charset=utf-8
content-length: 2471
etag: W/"9a7-hEXE7lJW5CDgD+e2FypGgChcgho"
date: Thu, 13 May 2021 19:20:46 GMT
x-envoy-upstream-service-time: 27
api-version: v1
server: istio-envoy
```

Similarly, we can use the `istioctl proxy-config listener [podname] -o json` to get the Envoy configuration and see where the Lua filter was added.

Here's a snippet from the above JSON configuration. Notice the Lua filter was injected just before the router filter:

```json
...
{
    "name": "envoy.lua",
    "typedConfig": {
        "@type": "type.googleapis.com/envoy.extensions.filters.http.lua.v3.Lua",
        "inlineCode": "function envoy_on_response(response_handle)\n  response_handle:headers():add(\"api-version\", \"v1\")\nend\n"
    }
},
{
    "name": "envoy.filters.http.router",
    "typedConfig": {
        "@type": "type.googleapis.com/envoy.extensions.filters.http.router.v3.Router"
    }
}
...
```

## Cleanup

To cleanup, delete the workloads and the EnvoyFilter:

```
kubectl delete -f envoy-demo-apps.yaml
kubectl delete -f envoy-header-filter.yaml
```