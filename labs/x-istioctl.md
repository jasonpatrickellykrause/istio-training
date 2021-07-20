# Istio CLI tools

Let's deploy the sample web-frontend and customers applications:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-frontend
  labels:
    app: web-frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-frontend
  template:
    metadata:
      labels:
        app: web-frontend
        version: v1
    spec:
      containers:
        - image: gcr.io/tetratelabs/web-frontend:1.0.0
          imagePullPolicy: Always
          name: web
          ports:
            - containerPort: 8080
          env:
            - name: CUSTOMER_SERVICE_URL
              value: 'http://customers.default.svc.cluster.local'
---
kind: Service
apiVersion: v1
metadata:
  name: web-frontend
  labels:
    app: web-frontend
spec:
  selector:
    app: web-frontend
  ports:
    - port: 80
      name: http
      targetPort: 8080
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: web-frontend
spec:
  hosts:
    - 'hello.com'
  gateways:
    - gateway
    - mesh
  http:
    - route:
        - destination:
            host: web-frontend.default.svc.cluster.local
            port:
              number: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: customers-v1
  labels:
    app: customers
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: customers
      version: v1
  template:
    metadata:
      labels:
        app: customers
        version: v1
    spec:
      containers:
        - image: gcr.io/tetratelabs/customers:1.0.0
          imagePullPolicy: Always
          name: svc
          ports:
            - containerPort: 3000
---
kind: Service
apiVersion: v1
metadata:
  name: customers
  labels:
    app: customers
spec:
  selector:
    app: customers
  ports:
    - port: 80
      name: http
      targetPort: 3000
---
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: gateway
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - 'hello.com'
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: customers
spec:
  hosts:
    - 'customers.default.svc.cluster.local'
  http:
    - route:
        - destination:
            host: customers.default.svc.cluster.local
            port:
              number: 80
            subset: v1
          weight: 80
        - destination:
            host: customers.default.svc.cluster.local
            port:
              number: 80
            subset: v2
          weight: 20
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: customers
spec:
  host: customers.default.svc.cluster.local
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: web-frontend
spec:
  host: web-frontend.default.svc.cluster.local
  subsets:
    - name: v1
      labels:
        version: v1
```

Save the above YAML to `samples.yaml` and create everything using `kubectl apply -f samples.yaml`.

## Proxy Status command

The `proxy status` command shows the discovery service sync status of all Envoys in a mesh.

For example, just running the command shows the list of all workloads in the mesh with the sync status for individual discovery services:

```sh
$ istioctl ps
NAME                                                   CDS        LDS        EDS        RDS          ISTIOD                      VERSION
customers-v1-796b6fc4bf-tzpwh.default                  SYNCED     SYNCED     SYNCED     SYNCED       istiod-56874696b5-7cg2r     1.10.0
istio-egressgateway-5b5b8cc687-zr7dw.istio-system      SYNCED     SYNCED     SYNCED     NOT SENT     istiod-56874696b5-7cg2r     1.10.0
istio-ingressgateway-5fd6f4747c-zcs9r.istio-system     SYNCED     SYNCED     SYNCED     SYNCED       istiod-56874696b5-7cg2r     1.10.0
web-frontend-74dd5cbcdc-vbt9r.default                  SYNCED     SYNCED     SYNCED     SYNCED       istiod-56874696b5-7cg2r     1.10.0
```

You can also specify a pod or deployment name as an argument, and in that case, you'll see the sync diff between istiod and the pod.

```sh
$ istioctl ps deployment/customers-v1
Clusters Match
Listeners Match
Routes Match (RDS last loaded at Sat, 26 Jun 2021 23:26:55 UTC)
```

## Proxy Config command

We can use the proxy config command to retrieve the different sections of the proxy configuration from the Envoy config dump.

### Listeners

Let's look at the listeners first:

```
$ istioctl proxy-config listener deployment/web-frontend
ADDRESS PORT MATCH DESTINATION
...
```

For the majority of commands, you can also use the `-o json` flag to get the same output in the JSON format, which is helpful if you want to look at all the details:

```
istioctl proxy-config listener deployment/web-frontend -o json
```

As the name suggests, the listener command shows all configured listeners for the specific pod, with the addresses, port numbers, matches, and destinations.

You can also filter the list down by the address field, port or type. For example, to see all listeners listening on port `80`, add the `--port` flag to the command: 

```
$ istioctl proxy-config listener deployment/web-frontend --port 80

ADDRESS PORT MATCH                        DESTINATION
0.0.0.0 80   Trans: raw_buffer; App: HTTP Route: 80
0.0.0.0 80   ALL                          PassthroughCluster
```

The `json` flag works as well in combination with the port filter, for example. So, if you want to get the `json` representation of all listeners listening on port `80`, add the output json flag to the command. Similarly, you can combine multiple flags - port and address, for example.

### Routes

The next sub-command is routes that list all routes in the configuration:

```
istioctl proxy-config routes deployment/web-frontend
```

It also supports filtering the routes based on the name. From the listeners command, we know the route is called "80", so we can now list all routes with that name using the `--name` flag:

```
$ istioctl proxy-config routes deployment/web-frontend --name 80
NAME     DOMAINS                               MATCH     VIRTUAL SERVICE
80       customers                             /*        customers.default
80       hello.com                             /*        web-frontend.default
80       istio-egressgateway.istio-system      /*
80       istio-ingressgateway.istio-system     /*
80       web-frontend                          /*
```

We can see the domain `customers`, the match, and the virtual service that configures the routing rules from the output. The virtual service name is read from the metadata that Istio sets in one of its filters: 

```json
"metadata": {
  "filterMetadata": {
      "istio": {
          "config": "/apis/networking.istio.io/v1alpha3/namespaces/default/virtual-service/web-frontend"
      }
  }
},
```

### Clusters

The next sub-command are clusters that show information about the cluster configuration for the Envoy instance in the pod:

```
istioctl pc clusters deployment/web-frontend
```

The clusters options support more flags to filter the list down. You can pretty much use any of the columns as a filter.

For example, the `direction` to show all outbound clusters:

```
istioctl pc clusters deployment/web-frontend --direction outbound
```

You'll notice the subsets and the destination rules that were created for specific services will show up as well. For example:

```
$ istioctl pc clusters deployment/web-frontend --direction outbound

SERVICE FQDN                                            PORT      SUBSET     DIRECTION     TYPE     DESTINATION RULE
customers.default.svc.cluster.local                     80        -          outbound      EDS      customers.default
customers.default.svc.cluster.local                     80        v1         outbound      EDS      customers.default
customers.default.svc.cluster.local                     80        v2         outbound      EDS      customers.default
...
```

### Endpoints

Finally, with the endpoints, you can get the list of all endpoints.

For example:

```
$ istioctl pc endpoints deployment/web-frontend
ENDPOINT                         STATUS      OUTLIER CHECK     CLUSTER
10.92.1.6:3000                   HEALTHY     OK                outbound|80|v1|customers.default.svc.cluster.local
10.92.1.6:3000                   HEALTHY     OK                outbound|80||customers.default.svc.cluster.local
10.92.2.11:8080                  HEALTHY     OK                outbound|80|v1|web-frontend.default.svc.cluster.local
10.92.2.11:8080                  HEALTHY     OK                outbound|80||web-frontend.default.svc.cluster.local
10.92.2.12:9090                  HEALTHY     OK                outbound|9090||prometheus.istio-system.svc.cluster.local
...
```

The output also shows the status of each endpoint, the outlier check (in case endpoint is ejected or not) and the cluster name that the endpoint belongs to. 

The output can also be filtered by the cluster name or the endpoint address.

### Secrets

The secrets command is still under development, but it will show you the information about the secret configuration and certs in the proxy:

```
$ istioctl pc secrets deployment/web-frontend
RESOURCE NAME     TYPE           STATUS     VALID CERT     SERIAL NUMBER                               NOT AFTER                NOT BEFORE
default           Cert Chain     ACTIVE     true           99793030141783437847211966546730179209      2021-06-28T21:37:17Z     2021-06-27T21:37:17Z
ROOTCA            CA             ACTIVE     true           105819247974076147373849192959783704407     2031-06-22T19:17:01Z     2021-06-24T19:17:01Z
```

The command will show the type of secrets as well as their expiry date.

### Log

Also, let's look at the log command. This command allows you to see and update log levels for each proxy individually.

For example, to list all loggers, run:

```
$ istioctl pc log deployment/web-frontend
active loggers:
  admin: warning
  aws: warning
  assert: warning
  backtrace: warning
  cache_filter: warning
...
```

Here's the full list of available logging scopes/loggers:

```
admin
aws
assert
backtrace
client
config
connection
conn_handler
dubbo
file
filter
forward_proxy
grpc
hc
health_checker
http
http2
hystrix
init
io
jwt
kafka
lua
main
misc
mongo
quic
pool
rbac
redis
router
runtime
stats
secret
tap
testing
thrift
tracing
upstream
udp
wasm
```

Then if you want to update the log levels, you can use the `[scope]:[log-level]` format. The log level can be one of the following values: `trace`, `debug`, `info`, `warning`, `error`, `critical`, or `off`.

For example, to set the `secret` scope log level to `debug` and `router` to `critical`, you'd run the following command:

```sh
istioctl pc log deployment/web-frontend --level secret:debug,router:critical
```

The command will list all loggers again with the updated log levels. Then, after you've gathered logs and did your investigation, you can revert to the default log levels using the `-r` flag:

```sh
istioctl pc log deployment/web-frontend -r
```

## Experimental commands

The commands under the experimental section are under active development and are not ready for production use - i.e., they might change or be removed.

### authz check

To show how the authz check command looks like, let's deploy a sample AuthorizationPolicy.

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-ingress-frontend
  namespace: default
spec:
  selector:
    matchLabels:
      app: web-frontend
  action: ALLOW
  rules:
    - from:
        - source:
            namespaces: ["istio-system"]
        - source:
            principals: ["istio-ingressgateway-service-account"]
```

Save the above YAML to `auth.yaml` and create it using `kubectl apply -f auth.yaml`.

The authz check command shows the authorization policies applied to the pods:

```
$ istioctl x authz check deployment/web-frontend
ACTION   AuthorizationPolicy              RULES
ALLOW    allow-ingress-frontend.default   1
```

### add-to-mesh/remove-from-mesh

You can use the add-to-mesh command to inject sidecar containers to already running deployments/pods. 

Let's start by creating a new namespace without Istio injection enabled:

```sh
kubectl create ns no-injection
```

Then we can create a new deployment called `httpbin` in the new namespace:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: httpbin
---
apiVersion: v1
kind: Service
metadata:
  name: httpbin
  labels:
    app: httpbin
    service: httpbin
spec:
  ports:
  - name: http
    port: 8000
    targetPort: 80
  selector:
    app: httpbin
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
      version: v1
  template:
    metadata:
      labels:
        app: httpbin
        version: v1
    spec:
      serviceAccountName: httpbin
      containers:
      - image: docker.io/kennethreitz/httpbin
        imagePullPolicy: IfNotPresent
        name: httpbin
        ports:
        - containerPort: 80
```
Save the above YAML to `httpbin.yaml` and deploy it to the `no-injection` namespace using `kubectl apply -f httpbin.yaml -n no-injection`.

We can now use the `add-to-mesh` command to inject the proxy into the deployment:

```sh
istioctl x add-to-mesh deployment httpbin -n no-injection
```

If you look at the pods, you'll notice that they've restarted and new pods startup with the sidecar proxy injected.

To remove the sidecar from any deployment or service, you can use the inverse command called `remove-from-mesh`:

```sh
istioctl x remove-from-mesh deployment httpbin -n no-injection
```

You can also use the add-to-mesh command to add external services running on VMs to the Istio service mesh. You can do that using the `external-service` sub-command, for example:

```sh
istioctl x add-to-mesh external-service [service-name] 1.1.1.1 http:9090 --labels app=my-service --serviceaccount my-service-sa
```

The above command will create the following:

- a Kubernetes service called `[service-name]` with the provider port name (`http`) and value (`9090`) and labels (`app=my-service`)

- a `MESH_INTERNAL` ServiceEntry called `mesh-expansion-[service-name]` with the endpoint `1.1.1.1`, host set to the Kubernetes service name and the same ports and labels as the service

### describe

The `describe` command describes the services/pods and their corresponding Istio configuration.

For example:

```
$ istioctl x describe service web-frontend
Service: web-frontend
   Port: http 80/HTTP targets pod port 8080
DestinationRule: web-frontend for "web-frontend.default.svc.cluster.local"
   Matching subsets: v1
      (Non-matching subsets v2)
   No Traffic Policy
VirtualService: web-frontend
   1 HTTP route(s)
RBAC policies: ns[default]-policy[allow-ingress-frontend]-rule[0]

Exposed on Ingress Gateway http://XX.XX.XXX.XX
VirtualService: web-frontend
   1 HTTP route(s)
```

The command shows any destination rules, virtual services, and RBAC policies applied to the pods targeted by the Kubernetes services. Additionally, the command will also show the ingress gateway IP address if the service is exposed through it.

### Workload entry and group commands

The `workload` command can help with configuring and deploying VM workloads.

You can use the `workload group` command to create the WorkloadGroup resource. For example:

```sh
$ istioctl x workload group create --name my-workload --namespace default --labels app=my-workload

apiVersion: networking.istio.io/v1alpha3
kind: WorkloadGroup
metadata:
  name: my-workload
  namespace: default
spec:
  metadata:
    annotations: {}
    labels:
      app: my-workload
  template:
    ports: {}
    serviceAccount: default
```
Once we've created the workload group, we can use the `workload entry configure` command to create all required configuration files for a workload instance that'll be running on a VM. 

```
istioctl x workload entry configure -f my-workloadgroup.yaml -o /some/folder
```

The above command will generate the following files in the `/some/folder`:

- `cluster.env`: contains the metadata describing the workload inside the cluster, such as namespace, service name, service account, trust domain, and so on.
- `hosts`: hosts that proxy on the VM will use to reach the istiod in the cluster. These entries will have to be added to the  `/etc/hosts` file on the VM
- `istio-token`: Kubernetes token used to get certs from the CA
- `mesh.yaml`: contains the configuration for the proxy, as well as the istiod discovery address
- `root-cert.pem`: root certificate used for authentication

## Cleanup

Run the following commands to delete all resources that were created  in this lab:

```sh
kubectl delete deployment web-frontend
kubectl delete svc web-frontend
kubectl delete vs web-frontend
kubectl delete dr web-frontend

kubectl delete deployment customers-v1
kubectl delete svc customers
kubectl delete vs customers
kubectl delete dr customers

kubectl delete gw gateway

kubectl delete authorizationpolicy allow-ingress-frontend

kubectl delete ns no-injection
```