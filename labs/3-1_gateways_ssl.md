# Lab: Gateway and SSL

In this lab we'll learn how to set up (self-signed) SSL certificate and configure Istio ingress gateway to use it.

## Deploying a sample application

We'll deploy a sample Hello world application:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: hello-world
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  labels:
    app: hello-world
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      serviceAccountName: hello-world
      containers:
        - image: gcr.io/tetratelabs/hello-world:1.0.0
          imagePullPolicy: Always
          name: svc
          ports:
            - containerPort: 3000
---
kind: Service
apiVersion: v1
metadata:
  name: hello-world
  labels:
    app: hello-world
spec:
  selector:
    app: hello-world
  ports:
    - port: 80
      name: http
      targetPort: 3000
```

Save the above to `hello-world.yaml` and deploy it using `kubectl apply -f hello-world.yaml`.

We also need to deploy a public gateway and a virtual service to expose the Hello world application through the ingress gateway:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: public-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - '*'
```

Save the above YAML to gateway.yaml and deploy it using kubectl apply -f gateway.yaml.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: helloworld
spec:
  hosts:
    - '*'
  gateways:
    - public-gateway
  http:
    - route:
      - destination:
          host: hello-world.default.svc.cluster.local
          port:
            number: 80
```

Save the above YAML to vs.yaml and deploy it using kubectl apply -f vs.yaml.

Finally, let's get the external IP address of the ingress gateway:

```sh
export INGRESS_IP=$(kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

If you run curl $INGRESS_IP you should get back a "Hello World" response.

## Using self-signed certificate

We'll manually create a self-signed certificate. First thing - pick a domain you want to use - note that to test this, you don’t have to own an actual domain name because we will use a self-signed certificate.

For this lab, I'll use `tetratelabs.dev` as my domain name and I'll use a subdomain called `hello`. We'll configure the gateway with a host called `hello.tetratelabs.dev` and present the self-signed certificate.

Let's also create a separate folder to hold the root certificate and the private key we will create:

```sh
mkdir -p tetratelabs-certs
```

### Create the cert files

Next we will create the root certificate called `tetratelabs.dev.crt` and the private key used for signing the certificate (file `tetratelabs.dev.key`):

```sh
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=tetratelabs.dev Inc./CN=tetratelabs.dev' -keyout tetratelabs.dev.key -out tetratelabs.dev.crt
```

The next step is to create the certificate signing request and the corresponding key:

```sh
openssl req -out hello.tetratelabs.dev.csr -newkey rsa:2048 -nodes -keyout hello.tetratelabs.dev.key -subj '/CN=hello.tetratelabs.dev/O=hello world from tetratelabs.dev'
```

Finally using the certificate authority and it's key as well as the certificate signing requests, we can create our own self-signed certificate:

```sh
openssl x509 -req -days 365 -CA tetratelabs.dev.crt -CAkey tetratelabs.dev.key -set_serial 0 -in hello.tetratelabs.dev.csr -out hello.tetratelabs.dev.crt
```

>If we'd use Let's Encrypt or SSL For Free, we'd get the `.crt` and `.key` from those services. The process of configuring the gateway would be identical from this point forward.

### Create the Kubernetes secret

Now that we have the certificate and the correspondig key we can create a Kubernetes secret to store them in our cluster.

We'll create the secret in the `istio-system` namespace and reference it from the Gateway resource:

```sh
kubectl create -n istio-system secret tls tetratelabs-credential --key=hello.tetratelabs.dev.key --cert=hello.tetratelabs.dev.crt
```

With secret in place, let's update the Gateway resource. Make sure you update the `hosts` field in case you used a different domain name:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: public-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        credentialName: tetratelabs-credential
      hosts:
        - hello.tetratelabs.dev
```

Save the above YAML to `gateway.yaml` and deploy it using `kubectl apply -f gateway.yaml`.

Let's also update the VirtualService to update the `hosts`field to match the `hosts` field in the Gateway:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: helloworld
spec:
  hosts:
    - hello.tetratelabs.dev
  gateways:
    - public-gateway
  http:
    - route:
      - destination:
          host: hello-world.default.svc.cluster.local
          port:
            number: 80
```

Save the above YAML to `vs.yaml` and deploy it using `kubectl apply -f vs.yaml`.

To try this out we'll use cURL and set the host header to `hello.tetratelabs.dev`. Then, we'll tell curl to resolve the `hello.tetratelabs.dev` to the `$INGRESS_IP` address. Additionaly, we are also presenting the certificate authority cert we created:

```sh
curl -v -H "Host:hello.tetratelabs.dev" --resolve "hello.tetratelabs.dev:443:$INGRESS_IP" --cacert tetratelabs.dev.crt "https://hello.tetratelabs.dev:443"
```

You'll notice the verbose output shows that the certificate was used:

```sh
...
Server certificate:
*  subject: CN=hello.tetratelabs.dev; O=hello world from tetratelabs.dev
*  start date: Dec 27 22:00:09 2021 GMT
*  expire date: Dec 27 22:00:09 2022 GMT
*  common name: hello.tetratelabs.dev (matched)
*  issuer: O=tetratelabs.dev Inc.; CN=tetratelabs.dev
*  SSL certificate verify ok.
...
```

# Cleanup

To remove all deployed resources, run:

```sh
kubectl delete deploy hello-world
kubectl delete svc hello-world
kubectl delete sa hello-world

kubectl delete gw public-gateway
kubectl delete vs helloworld
kubectl delete secret tetratelabs-credential -n istio-system
```






