# Wasm Extensions

We will be using [TinyGo](https://tinygo.org), [proxy-wasm-go-sdk](https://github.com/tetratelabs/proxy-wasm-go-sdk) and [func-e CLI](https://func-e.io) to build and test an Envoy Wasm extension. Then we'll show a way to configure the Wasm module using the EnvoyFilter resource and deploy it to Envoy sidecars in a Kubernetes cluster.

We'll start with something trivial for our first example and write a simple Wasm module using TinyGo that adds a custom header to response headers.

## Installing func-e CLI

Let's get started by downloading func-e CLI and installing it to `/usr/local/bin`:

```sh
curl https://func-e.io/install.sh | sudo bash -s -- -b /usr/local/bin
```

Once downloaded, let's run it to make sure all is good:

```sh
$ func-e --version
func-e version 1.1.1
```

## Installing TinyGo

TinyGo powers the SDK we'll be using as Wasm doesn't support the official Go compiler. 

Let's download and install the TinyGo:

```sh
wget https://github.com/tinygo-org/tinygo/releases/download/v0.21.0/tinygo_0.21.0_amd64.deb
sudo dpkg -i tinygo_0.21.0_amd64.deb
```

You can run `tinygo version` to check the installation is successful:

```sh
$ tinygo version
tinygo version 0.21.0 linux/amd64 (using go version go1.17.2 and LLVM version 11.0.0)
```

## Scaffolding the Wasm module

We'll start by creating a new folder for our extension, initializing the Go module, and downloading the SDK dependency:

```sh
$ mkdir header-filter && cd header-filter
$ go mod init header-filter
$ go mod edit -require=github.com/tetratelabs/proxy-wasm-go-sdk@main
$ go mod download github.com/tetratelabs/proxy-wasm-go-sdk
```

Next, let's create the `main.go` file where the code for our WASM extension will live:

```go
package main

import (
  "github.com/tetratelabs/proxy-wasm-go-sdk/proxywasm"
  "github.com/tetratelabs/proxy-wasm-go-sdk/proxywasm/types"
)

func main() {
  proxywasm.SetVMContext(&vmContext{})
}

type vmContext struct {
  // Embed the default VM context here,
  // so that we don't need to reimplement all the methods.
  types.DefaultVMContext
}

// Override types.DefaultVMContext.
func (*vmContext) NewPluginContext(contextID uint32) types.PluginContext {
  return &pluginContext{}
}

type pluginContext struct {
  // Embed the default plugin context here,
  // so that we don't need to reimplement all the methods.
  types.DefaultPluginContext
}

// Override types.DefaultPluginContext.
func (*pluginContext) NewHttpContext(contextID uint32) types.HttpContext {
  return &httpHeaders{contextID: contextID}
}

type httpHeaders struct {
  // Embed the default http context here,
  // so that we don't need to reimplement all the methods.
  types.DefaultHttpContext
  contextID uint32
}

func (ctx *httpHeaders) OnHttpRequestHeaders(numHeaders int, endOfStream bool) types.Action {
  proxywasm.LogInfo("OnHttpRequestHeaders")
  return types.ActionContinue
}

func (ctx *httpHeaders) OnHttpResponseHeaders(numHeaders int, endOfStream bool) types.Action {
  proxywasm.LogInfo("OnHttpResponseHeaders")
  return types.ActionContinue
}

func (ctx *httpHeaders) OnHttpStreamDone() {
  proxywasm.LogInfof("%d finished", ctx.contextID)
}
```

Save the above contents to a file called `main.go`.

Let's build the filter to check everything is good:

```sh
tinygo build -o main.wasm -scheduler=none -target=wasi main.go
```

The build command should run successfully, and it should generate a file called `main.wasm`.

We'll use `func-e` to run a local Envoy instance to test the extension we've built.

First, we need an Envoy config that will configure the extension:

```yaml
static_resources:
  listeners:
    - name: main
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 18000
      filter_chains:
        - filters:
            - name: envoy.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: ingress_http
                codec_type: auto
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: local_service
                      domains:
                        - "*"
                      routes:
                        - match:
                            prefix: "/"
                          direct_response:
                            status: 200
                            body:
                              inline_string: "hello world\n"
                http_filters:
                  - name: envoy.filters.http.wasm
                    typed_config:
                      "@type": type.googleapis.com/udpa.type.v1.TypedStruct
                      type_url: type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm
                      value:
                        config:
                          vm_config:
                            runtime: "envoy.wasm.runtime.v8"
                            code:
                              local:
                                filename: "main.wasm"
                  - name: envoy.filters.http.router

admin:
  access_log_path: "/dev/null"
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 8001
```

Save the above to `envoy.yaml` file.

The Envoy configuration sets up a single listener on port 18000 that returns a direct response (HTTP 200) with body `hello world`. Inside the `http_filters` section, we're configuring the `envoy.filters.http.wasm` filter and referencing the local WASM file (`main.wasm`) we've built earlier.

Let's run the Envoy with this configuration in the background:

```sh
func-e run -c envoy.yaml &
```

Envoy instance should start without any issues. Once it's started, we can send a request to the port Envoy is listening on (`18000`):

```sh
$ curl localhost:18000
[2021-11-04 22:41:19.982][91521][info][wasm] [source/extensions/common/wasm/context.cc:1167] wasm log: OnHttpRequestHeaders
[2021-11-04 22:41:19.982][91521][info][wasm] [source/extensions/common/wasm/context.cc:1167] wasm log: OnHttpResponseHeaders
[2021-11-04 22:41:19.983][91521][info][wasm] [source/extensions/common/wasm/context.cc:1167] wasm log: 2 finished
hello world
```

The output shows the two log entries - one from the OnHttpRequestHeaders handler and the second one from the OnHttpResponseHeaders handler. The last line is the example response returned by the direct response configuration in the filter.

You can stop the proxy by bringing the process to the foreground with `fg` and pressing CTRL+C to stop it.

## Setting additional headers on HTTP response

Let's open the `main.go` file and add a header to the response headers. We'll be updating the OnHttpResponseHeaders function to do that.

We'll call the `AddHttpResponseHeader` function to add a new header. Update the OnHttpResponseHeaders function to look like this: 

```go
func (ctx *httpHeaders) OnHttpResponseHeaders(numHeaders int, endOfStream bool) types.Action {
  proxywasm.LogInfo("OnHttpResponseHeaders")
  err := proxywasm.AddHttpResponseHeader("my-new-header", "some-value-here")
  if err != nil {
    proxywasm.LogCriticalf("failed to add response header: %v", err)
  }
  return types.ActionContinue
}
```

Let's rebuild the extension:

```sh
tinygo build -o main.wasm -scheduler=none -target=wasi main.go
```

And we can now re-run the Envoy proxy with the updated extension (make sure to run `fg` first to bring the previous Envoy instance to the foreground and then use <kbd>CTRL+C</kbd> to stop it):

```sh
func-e run -c envoy.yaml &
```

Now, if we send a request again (make sure to add the `-v` flag), we'll see the header that got added to the response:

```sh
$ curl -v localhost:18000
...
< HTTP/1.1 200 OK
< content-length: 13
< content-type: text/plain
< my-new-header: some-value-here
< date: Mon, 22 Jun 2021 17:02:31 GMT
< server: envoy
<
hello world
```

You can stop running Envoy by typing `fg` and then pressing <kbd>CTRL+C</kbd>.

## Reading values from configuration

Hardcoding values like that in code is never a good idea. Let's see how we can read the additional headers.

1. Add the `additionalHeaders` and `contextID` to the `pluginContext` struct:

  ```go
  type pluginContext struct {
    // Embed the default plugin context here,
    // so that we don't need to reimplement all the methods.
    types.DefaultPluginContext
    additionalHeaders map[string]string
    contextID         uint32
  }
  ```

2. Update the `NewPluginContext` function to initialize the values:

  ```go
  func (*vmContext) NewPluginContext(contextID uint32) types.PluginContext {
    return &pluginContext{contextID: contextID, additionalHeaders: map[string]string{}}
  }
  ```
  
3. We can add the `OnPluginStart` function where we read in values from the Envoy configuration and store the key/value pairs in the `additionalHeaders` map:

  ```go
  func (ctx *pluginContext) OnPluginStart(pluginConfigurationSize int) types.OnPluginStartStatus {
    // Get the plugin configuration
    config, err := proxywasm.GetPluginConfiguration()
    if err != nil && err != types.ErrorStatusNotFound {
      proxywasm.LogCriticalf("failed to load config: %v", err)
      return types.OnPluginStartStatusFailed
    }

    // Read the config
    scanner := bufio.NewScanner(bytes.NewReader(config))
    for scanner.Scan() {
      line := scanner.Text()
      if strings.HasPrefix(line, "#") {
        continue
      }
      // Each line in the config is in the "key=value" format
      if tokens := strings.Split(scanner.Text(), "="); len(tokens) == 2 {
        ctx.additionalHeaders[tokens[0]] = tokens[1]
      }
    }
    return types.OnPluginStartStatusOK
  }
  ```

To access the configuration values we've set, we need to add the map to the HTTP context when we initialize it. To do that, we need to update the `httpheaders` struct first:

```go
type httpHeaders struct {
  // Embed the default http context here,
  // so that we don't need to reimplement all the methods.
  types.DefaultHttpContext
  contextID         uint32
  additionalHeaders map[string]string
}
```

Then, in the `NewHttpContext` function we can instantiate the httpHeaders with the additional headers map coming from the plugin context:

```go
func (ctx *pluginContext) NewHttpContext(contextID uint32) types.HttpContext {
  return &httpHeaders{contextID: contextID, additionalHeaders: ctx.additionalHeaders}
}
```

Finally, in order to set the headers we modiy the `OnHttpResponseHeaders` function, iterate through the `additionalHeaders` map and call the `AddHttpResponseHeader` for each item:

```go
func (ctx *httpHeaders) OnHttpResponseHeaders(numHeaders int, endOfStream bool) types.Action {
  proxywasm.LogInfo("OnHttpResponseHeaders")

  for key, value := range ctx.additionalHeaders {
    if err := proxywasm.AddHttpResponseHeader(key, value); err != nil {
        proxywasm.LogCriticalf("failed to add header: %v", err)
        return types.ActionPause
    }
    proxywasm.LogInfof("header set: %s=%s", key, value)
  }
    
  return types.ActionContinue
}
```

Let's rebuild the extension again:

```sh
tinygo build -o main.wasm -scheduler=none -target=wasi main.go
```

Also, let's update the config file to include additional headers in the filter configuration (the `configuration` field):

```yaml
- name: envoy.filters.http.wasm
  typed_config:
    "@type": type.googleapis.com/udpa.type.v1.TypedStruct
    type_url: type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm
    value:
      config:
        vm_config:
          runtime: "envoy.wasm.runtime.v8"
          code:
            local:
              filename: "main.wasm"
        # ADD THESE LINES
        configuration:
          "@type": type.googleapis.com/google.protobuf.StringValue
          value: |
            header_1=somevalue
            header_2=secondvalue
```

With the filter updated, we can re-run the proxy again. When you send a request, you'll notice the headers we set in the filter configuration are added as response headers:

```sh
$ curl -v localhost:18000
...
< HTTP/1.1 200 OK
< content-length: 13
< content-type: text/plain
< header_1: somevalue
< header_2: secondvalue
< date: Mon, 22 Jun 2021 17:54:53 GMT
< server: envoy
...
```

## Add a metric

Let's add another feature - a counter that increases each time there's a request header called `hello` set.

First, let's update the `pluginContext` to include the `helloHeaderCounter`:

```go
type pluginContext struct {
  // Embed the default plugin context here,
  // so that we don't need to reimplement all the methods.
  types.DefaultPluginContext
  additionalHeaders  map[string]string
  contextID          uint32
  // ADD THIS LINE
  helloHeaderCounter proxywasm.MetricCounter 
}
```

With the metric counter in the struct, we can now create it in the `NewPluginContext` function. We'll call the header `hello_header_counter`.

```go
func (*vmContext) NewPluginContext(contextID uint32) types.PluginContext {
  return &pluginContext{contextID: contextID, additionalHeaders: map[string]string{}, helloHeaderCounter: proxywasm.DefineCounterMetric("hello_header_counter")}
}
```

Since we want need to check the incoming request headers to decide whether to increment the counter, we need to add the `helloHeaderCounter` to the `httpHeaders` struct as well:

```go
type httpHeaders struct {
  // Embed the default http context here,
  // so that we don't need to reimplement all the methods.
  types.DefaultHttpContext
  contextID          uint32
  additionalHeaders  map[string]string
  // ADD THIS LINE
  helloHeaderCounter proxywasm.MetricCounter
}
```

Also, we need to get the counter from the `pluginContext` and set it when we're creating the new HTTP context:

```go
// Override types.DefaultPluginContext.
func (ctx *pluginContext) NewHttpContext(contextID uint32) types.HttpContext {
  return &httpHeaders{contextID: contextID, additionalHeaders: ctx.additionalHeaders, helloHeaderCounter: ctx.helloHeaderCounter}
}
```

Now that we've piped the `helloHeaderCounter` all the way through to the `httpHeaders`, we can use it in the `OnHttpRequestHeaders` function:

```go
func (ctx *httpHeaders) OnHttpRequestHeaders(numHeaders int, endOfStream bool) types.Action {
  proxywasm.LogInfo("OnHttpRequestHeaders")

  _, err := proxywasm.GetHttpRequestHeader("hello")
  if err != nil {
    // Ignore if header is not set
    return types.ActionContinue
  }

  ctx.helloHeaderCounter.Increment(1)
  proxywasm.LogInfo("hello_header_counter incremented")
  return types.ActionContinue
}
```

Here, we're checking if the "hello" request header is defined (note that we don't care about the header value), and if it's defined, we call the `Increment` function on the counter instance. Otherwise, we'll ignore it and return ActionContinue if we get an error from the `GetHttpRequestHeader` call.

Let's rebuild the extension again:

```sh
tinygo build -o main.wasm -scheduler=none -target=wasi main.go
```

And then re-run the Envoy proxy. Make a couple of requests like this:

"`sh
curl -H "hello: something" localhost:18000
```

You'll notice the log Envoy log entry like this one:

"`text
wasm log: hello_header_counter incremented
```

You can also use the admin address on port 8001 to check that the metric is being tracked:

```sh
$ curl localhost:8001/stats/prometheus | grep hello
# TYPE envoy_hello_header_counter counter
envoy_hello_header_counter{} 1
```

## Deploying Wasm module to Istio using WasmPlugin

The resource we can use to deploy a Wasm module to Istio is called the WasmPlugin. The WasmPlugin allows us to select the workloads we want to apply the Wasm module to as well as point to the Wasm module.

In previous Istio versions we'd have to use the EnvoyFilter resource to configure the Wasm plugins. Within the resource we could either point to a local Wasm file (i.e. file that's accessible by the Istio proxy) or a remote location. Using the remote location (e.g. `http://some-storage-account/main.wasm`), the Istio proxy would download the Wasm file and cache it in the volume accessible to the proxy.

With the move to the WasmPlugin resource, a feature was added that enables Istio proxy (or istio-agent to be precise) download the Wasm file from an OCI-compliant registry. What that means is that we can treate the Wasm files just like we treat Docker images. We can push them to a registry, version them using tags and reference them from the WasmPlugin resource.

In the previous example, there was no need to push or publish the `main.wasm` file anywhere, as it was accessible by the Envoy proxy because everything was running locally. However, now that we want to run the Wasm module in Envoy proxies that are part of the Istio service mesh, we need to make the `main.wasm` file available to all those proxies so they can load and run it.

Since we'll be building and pushing the Wasm file we'll need a very minimal Dockerfile in the project:

```dockerfile
FROM scratch
COPY main.wasm ./plugin.wasm
```

This Docker file just copies the `main.wasm` file to the container as `plugin.wasm` file. Save the above contents to `Dockerfile`.

Before we build the image, let's create a repository - replace the `REPOSITORY_NAME` with your username:

```sh
gcloud artifacts repositories create REPOSITORY_NAME --repository-format=docker --location=us-west1 --description="Docker repository for Istio training"
```

And configure the credentials:

```sh
gcloud auth configure-docker us-west1-docker.pkg.dev
```

We can now use the full repository name to build the image called `wasm` (make sure to replace the `YOUR_PROJECT_NAME`, `REPOSITORY_NAME`, and the region if you used a different one):

```sh
docker build -t us-west1-docker.pkg.dev/[YOUR_PROJECT_NAME]/[REPOSITORY_NAME]/wasm:v1 .
```

Next, let's push the image to the registry, so the Istio agent can pull it when needed:

```sh
docker push us-west1-docker.pkg.dev/[YOUR_PROJECT_NAME]/[REPOSITORY_NAME]/wasm:v1
```

Finally, we'll update the policy to allow public access to the repository - this will allow Istio to pull the images without any extra credentials. Note that the WasmPlugin resource also supports providing the image pull secrets file for private registries.

The following command IaM policy binding for the repository and assigns all users the `roles/viewer` role: 

```sh
gcloud artifacts repositories add-iam-policy-binding [REPOSITORY_NAME] --location=us-west1 --member=allUsers --role=roles/viewer
```  

We can now create the WasmPlugin resource that tells Envoy where to download the extension from as well as to which workloads to apply it to (we'll use `httpbin` workload we'll deploy next). Also make sure you update the `YOUR_PROJECT_NAME` and `REPOSITORY_NAME` values in the `url` field:

```yaml
apiVersion: extensions.istio.io/v1alpha1
kind: WasmPlugin
metadata:
  name: wasm-example
  namespace: default
spec:
  selector:
    matchLabels:
      app: httpbin
  url: oci://us-west1-docker.pkg.dev/[YOUR_PROJECT_NAME]/[REPOSITORY_NAME]/wasm:v1
```

Save the above YAML to `plugin.yaml` and deploy it using `kubectl apply -f plugin.yaml`.

To try out the module, you can deploy a sample workload. We'll use the `httpbin` example:

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
  template:
    metadata:
      labels:
        app: httpbin
    spec:
      serviceAccountName: httpbin
      containers:
      - image: docker.io/kennethreitz/httpbin
        imagePullPolicy: IfNotPresent
        name: httpbin
        ports:
        - containerPort: 80
```

Save the above file to `httpbin.yaml` and deploy it using `kubectl apply -f httpbin.yaml`. 

Before continuing, check that the httpbin Pod is up and running:

```sh
$ kubectl get po
NAME                       READY   STATUS        RESTARTS   AGE
httpbin-66cdbdb6c5-4pv44   2/2     Running       1          11m
```

To see if something went wrong with downloading the Wasm module, you can look at the istiod logs.

Let's try out the deployed Wasm module! 

We will create a single Pod inside the cluster, and from there, we will send a request to `http://httpbin:8000/get` and include the `hello` header.

```sh
$ kubectl run curl --image=curlimages/curl -it --rm -- /bin/sh
Defaulted container "curl" out of: curl, istio-proxy, istio-init (init)
If you don't see a command prompt, try pressing enter.
/ $
```

Once you get the prompt to the curl container, send a request to the `httpbin` service:

```sh
/ $ curl -v -H "hello: value" http://httpbin:8000/headers
> GET /headers HTTP/1.1
> User-Agent: curl/7.35.0
> Host: httpbin:8000
> Accept: */*
>
< HTTP/1.1 200 OK
< server: envoy
< date: Mon, 22 Jun 2021 18:52:17 GMT
< content-type: application/json
< content-length: 525
< access-control-allow-origin: *
< access-control-allow-credentials: true
< x-envoy-upstream-service-time: 3
...
```


If we exit the pod and look at the stats, we'll notice that the `hello_header_counter` has increased:

```
kubectl exec -it [httpbin-pod-name] -c istio-proxy -- curl localhost:15000/stats/prometheus | grep hello

# TYPE hello_header_counter counter
hello_header_counter{} 1
```

## Cleanup

To delete all resource created during this lab, run the following:

```sh
kubectl delete wasmplugin wasm-example
kubectl delete deployment httpbin
kubectl delete svc httpbin
kubectl delete sa httpbin
```