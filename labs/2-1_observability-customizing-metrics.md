# Lab: Customizing metrics

Let's look at the metrics that are being collected by the mesh. We'll use a simple Nginx deployment:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    app: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    app: my-nginx
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      app: my-nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: my-nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

Save the above YAML to `my-nginx.yaml` and create the deployment using `kubectl apply -f my-nginx.yaml`.

Now we can run `kubectl get services` and get the external IP address of the `my-nginx` service. Note that the load balancer creation might take a couple of minutes. During that time the value in the `EXTERNAL-IP` column will be `<pending>`.

```sh
$ kubectl get svc
NAME         TYPE           CLUSTER-IP   EXTERNAL-IP      PORT(S)        AGE
kubernetes   ClusterIP      10.48.0.1    <none>           443/TCP        73m
my-nginx     LoadBalancer   10.48.0.94   [IP HERE]   80:31191/TCP   4m6s
```

Once the IP address is available, let’s store it as an environment variable so we can use it throughout this lab:

```sh
export NGINX_IP=$(kubectl get service my-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

You can now run curl against the above IP and you should get back the default Nginx page:

```sh
$ curl $NGINX_IP
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

Alternatively, you can open the IP address in the browser to see the default Nginx page.

Use curl to send a couple of requests to the Nginx deployment to generate some data for the metrics.

Next, we can use the `/stats/prometheus` endpoint on the `istio-proxy` container and look at the `istio_requests_total` metric:

```sh
$ kubectl exec -it deploy/my-nginx -c istio-proxy -- curl localhost:15000/stats/prometheus | grep istio_requests_total
# TYPE istio_requests_total counter
istio_requests_total{response_code="200",reporter="destination",source_workload="unknown",source_workload_namespace="unknown",source_principal="unknown",source_app="unknown",source_version="unknown",source_cluster="unknown",destination_workload="my-nginx",destination_workload_namespace="default",destination_principal="unknown",destination_app="my-nginx",destination_version="unknown",destination_service="my-nginx.default.svc.cluster.local",destination_service_name="my-nginx",destination_service_namespace="default",destination_cluster="Kubernetes",request_protocol="http",response_flags="-",grpc_response_status="",connection_security_policy="none",source_canonical_service="unknown",destination_canonical_service="my-nginx",source_canonical_revision="latest",destination_canonical_revision="latest"} 9
```

Since we have Prometheus installed we can also open the Prometheus dashboard and look at the metrics there (use `istioctl dash prometheus`). Then, we can search for the `istio_request_duration_milliseconds_bucket` for example to get one of the distribution metrics. Note that there are 3 different metrics for the request duration, the bucket, count and sum.

### Adding a dimension to existing metric

Let's use an example where we add a new dimension called `app_containers` to the `istio_requests_total` metric.

To do that, we'll create the Telemetry API resource that adds the dimension to all Prometheus metrics to all workloads in the `default` namespace:

```yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: app-containers-dimension
  namespace: default
spec:
  metrics:
  - providers:
    - name: prometheus
    overrides:
    - match:
        metric: REQUEST_COUNT
      tagOverrides:
        app_containers:
          value: "node.metadata['APP_CONTAINERS']"
```

Save the above YAML to `app-containers-dimension.yaml` and deploy it using `kubectl apply -f app-containers-dimension.yaml`.

Because the `app_containers` is not in the list of default stat tags, we need to include it. The way to do that is by either updating the IstioOperator and doing it mesh-wide or adding an annotation to the Pod spec.

Let's edit the my-nginx deployment (`kubectl edit deploy my-nginx`) and add the following annotation - make sure you add the annotation to the Pod spec:

```yaml
...
template:
  metadata:
    annotations:
      sidecar.istio.io/extraStatTags: app_containers
```

Save the changes and wait for the Pod to be restarted. Once the Pod restarts, make a couple of requests to the $NGINX_IP and then look at the metrics:

```sh
$ kubectl exec -it deploy/my-nginx -c istio-proxy -- curl localhost:15000/stats/prometheus | grep app_containers
istio_requests_total{response_code="200",reporter="destination",source_workload="unknown",source_workload_namespace="unknown",source_principal="unknown",source_app="unknown",source_version="unknown",source_cluster="unknown",destination_workload="my-nginx",destination_workload_namespace="default",destination_principal="unknown",destination_app="my-nginx",destination_version="unknown",destination_service="my-nginx.default.svc.cluster.local",destination_service_name="my-nginx",destination_service_namespace="default",destination_cluster="Kubernetes",request_protocol="http",response_flags="-",grpc_response_status="",connection_security_policy="none",source_canonical_service="unknown",destination_canonical_service="my-nginx",source_canonical_revision="latest",destination_canonical_revision="latest",app_containers="nginx"} 13
```

You'll notice the `app_containers="nginx"` attribute was added to the existing metric.

### Creating a new metric

Creating a new metric is similar to updating the existing one - we need to create an EnvoyFilter to define the metric and then register it via an annotation or IstioOperator.

We define the new metric in the same istio.stats filter configuration where we added the dimensions, but under the `definitions` field:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: new-envoy-metric
spec:
  configPatches:
  - applyTo: HTTP_FILTER
    match:
      context: ANY
      listener:
        filterChain:
          filter:
            name: envoy.filters.network.http_connection_manager
            subFilter:
              name: envoy.filters.http.router
      proxy:
        proxyVersion: ^1\.12.*
    patch:
      operation: INSERT_BEFORE
      value:
        name: istio.stats
        typed_config:
          '@type': type.googleapis.com/udpa.type.v1.TypedStruct
          type_url: type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm
          value:
            config:
              configuration:
                '@type': type.googleapis.com/google.protobuf.StringValue
                value: |
                  {
                    "debug": "false",
                    "stat_prefix": "istio",
                    "definitions": [
                      {
                        "name": "simple_counter",
                        "type": "COUNTER",
                        "value": "1"
                      }
                    ]
                  }
              root_id: stats_outbound
              vm_config:
                code:
                  local:
                    inline_string: envoy.wasm.stats
                runtime: envoy.wasm.runtime.null
                vm_id: stats_outbound
```

With the above YAML we're defining a new counter metric called `simple_counter`. Note that the value is a string and we could use an expression there to define when to increment the counter.

Save the YAML to `new-metric.yaml` and create it using `kubectl apply -f new-metric.yaml`. 

Just like we did before, we need to edit the my-nginx deployment and add the annotation to register this new metric.

Run `kubectl edit deploy my-nginx` and add the following annotation to the Pod template, right after the annotation we added earlier:

```yaml
sidecar.istio.io/statsInclusionPrefixes: istio_simple_counter
```

Note that we have to prefix the metric name with `istio`, because that's the stat prefix defined in the Envoy filter. Save the deployment, make a couple of requests to $NGINX_IP and then check the metrics:

```
$ kubectl exec -it deploy/my-nginx -c istio-proxy -- curl localhost:15000/stats/prometheus | grep simple
# TYPE istio_simple_counter counter
istio_simple_counter{} 4
```

## Cleanup

To remove the Nginx application, run:

```sh
kubectl delete deploy my-nginx
kubectl delete svc my-nginx
```

To remove the EnvoyFilter and Telemetry resource, run:

```sh
kubectl delete telemetry app-containers-dimension
kubectl delete envoyfilter new-envoy-metric
```