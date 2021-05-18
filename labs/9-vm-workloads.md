# Connecting a VM to Istio service mesh

In this lab, we will learn how to connect a workload running on a virtual machine to the Istio service mesh running on a Kubernetes cluster. Both Kubernetes cluster and the virtual machine will be running on Google Cloud Platform (GCP).

After we've created a Kubernetes cluster, we can download, install Istio, and configure Istio. We are using a pre-alpha feature that automatically creates WorkloadEntries for our VMs. To support this, we have to set the `PILOT_ENABLE_WORKLOAD_ENTRY_AUTOREGISTRATION` variable to true when installing Istio on the Kubernetes cluster.

## Installing Istio on a Kubernetes cluster

Let's download Istio 1.9.5:

```bash
$ curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.9.5 sh -
```

With Istio downloaded, we can install it using the default profile with automatic WorkloadEntry creation:

```bash
istioctl install --set values.pilot.env.PILOT_ENABLE_WORKLOAD_ENTRY_AUTOREGISTRATION=true
```

Next, we will deploy a separate gateway that we will use to expose the Istio's control plane to the virtual machine. The Istio package we downloaded contains a script we can use to generate the YAML that will deploy an Istio operator that creates the new gateway called `istio-eastwestgateway`.

Go to the folder where you downloaded Istio (e.g. `istio-1.9.5`) and run this script:

```bash
samples/multicluster/gen-eastwest-gateway.sh --single-cluster | istioctl install -y -f -
```

You can list the Pods in the `istio-system` namespaces to check the gateway that was installed:

```bash
$ kubectl get po -n istio-system
NAME                                     READY   STATUS    RESTARTS   AGE
istio-eastwestgateway-59cc6fcdb6-b64rs   1/1     Running   0          34s
istio-ingressgateway-69b7dd67d6-5pj6b    1/1     Running   0          7m58s
istiod-58596b9585-rq8f5                  1/1     Running   0          8m11s
```

For virtual machines to access the Istio's control plane, we need to create a Gateway resource to configure the `istio-eastwestgateway`, and a VirtualService that has the Gateway attached. 

We can use another script from the Istio package to create these resources and expose the control plane:

```
$ kubectl apply -f samples/multicluster/expose-istiod.yaml
gateway.networking.istio.io/istiod-gateway created
virtualservice.networking.istio.io/istiod-vs created
```

## Preparing virtual machine namespace and files

For the virtual machine workloads, we have to create a separate namespace to store the WorkloadEntry resource and any other VM workload related resources. Additionally, we will have to export the cluster environment file, token, certificate, and other files we will have to transfer to the virtual machine.

Let's create a separate folder called `vm-files` to store these files. We can also save the full path to the folder in the `WORK_DIR` environment variable:

```
$ mkdir -p vm-files
$ export WORK_DIR="$PWD/vm-files"
```

We also set a couple of environment variables before continuing, so we don't have to re-type the values each time:

```
export VM_APP="hello-vm"
export VM_NAMESPACE="vm-namespace"
export SERVICE_ACCOUNT="vm-sa"
```

Let's create the VM namespace and the service account we will use for VM workloads in the same namespace:

```bash
$ kubectl create ns "${VM_NAMESPACE}"
namespace/vm-namespace created

$ kubectl create serviceaccount "${SERVICE_ACCOUNT}" -n "${VM_NAMESPACE}"
serviceaccount/vm-sa created
```

We can now create a WorkloadGroup resource using Istio CLI. Run the command to save the WorkloadGroup YAML to `workloadgroup.yaml`

```
$ istioctl x workload group create --name "${VM_APP}" --namespace "${VM_NAMESPACE}" --labels app="${VM_APP}" --serviceAccount "${SERVICE_ACCOUNT}" > workloadgroup.yaml
```

Here's how the contents of the `workloadgroup.yaml` should look like:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: WorkloadGroup
metadata:
  name: hello-vm
  namespace: vm-namespace
spec:
  metadata:
    annotations: {}
    labels:
      app: hello-vm
  template:
    ports: {}
    serviceAccount: vm-sa
```

We can now deploy this WorkloadGroup to the virtual machine namespace:

```
$ kubectl apply -f workloadgroup.yaml -n ${VM_NAMESPACE}
workloadgroup.networking.istio.io/hello-vm created
```

Virtual machine needs information about the cluster and Istio's control plane to connect to it. To generate the required files, we can run `istioctl x workload entry` command. We save all generated files to the `WORK_DIR`:

```bash
$ istioctl x workload entry configure -f workloadgroup.yaml -o "${WORK_DIR}" --autoregister
warning: a security token for namespace vm-namespace and service account vm-sa has been generated and stored at /vm-files/istio-token
configuration generation into directory /vm-files was successful
```

## Configuring the virtual machine

Now it's time to create and configure a virtual machine. I am running the virtual machine in GCP, just like the Kubernetes cluster. The virtual machine is using the Debian GNU/Linux 10 (Buster) image. Make sure you check "Allow HTTP traffic" under the Firewall section and you have SSH access to the instance.

>In this example, we run a simple Python HTTP server on port 80. You could configure any other service on a different port. Just make sure you configure the security and firewall rules accordingly.

From the instance details page, click the SSH dropdown and select "View gcloud command". You can run that command to set up the SSH connection with the instance (i.e. create the SSH keys).

1. Copy the files from `vm-files` folder to the home folder on the instance. Replace `USERNAME` and `INSTANCE_IP` accordingly.


    ```bash
    $ scp vm-files/* [USERNAME]@[INSTANCE_IP]:~
    Enter passphrase for key '/Users/peterj/.ssh/id_rsa':
    bash: warning: setlocale: LC_ALL: cannot change locale (en_US.UTF-8)
    cluster.env                                          100%  589    12.6KB/s   00:00
    hosts                                                100%   38     0.8KB/s   00:00
    istio-token                                          100%  906    19.4KB/s   00:00
    mesh.yaml                                            100%  667    14.4KB/s   00:00
    root-cert.pem                                        100% 1094    23.5KB/s   00:00
    ```
     
    Alternatively, you an use `gcloud` command to copy the files over: 
    ```sh
    gcloud compute scp vm-files/* [INSTANCE_NAME]:~ --zone=[INSTANCE_ZONE]
    ```

2. SSH into the instance and copy the root certificate to `/etc/certs` (you can use the comamnd from the instance details page in the SSH dropdown):

    ```bash
    sudo mkdir -p /etc/certs
    sudo cp root-cert.pem /etc/certs/root-cert.pem
    ```

3. Copy the `istio-token` file to `/var/run/secrets/tokens` folder:

    ```bash
    sudo mkdir -p /var/run/secrets/tokens
    sudo cp istio-token /var/run/secrets/tokens/istio-token
    ```

4. Download and install the Istio sidecar package:

    ```bash
    curl -LO https://storage.googleapis.com/istio-release/releases/1.9.5/deb/istio-sidecar.deb
    sudo dpkg -i istio-sidecar.deb
    ```

5. Copy `cluster.env` to `/var/lib/istio/envoy/`:

    ```bash
    sudo cp cluster.env /var/lib/istio/envoy/cluster.env
    ```

6. Copy Mesh config (`mesh.yaml`) to `/etc/istio/config/mesh`:

    ```bash
    sudo cp mesh.yaml /etc/istio/config/mesh
    ```

7. Add the istiod host to the `/etc/hosts` file:

    ```bash
    sudo sh -c 'cat hosts >> /etc/hosts'
    ```

8. Change the ownership of files in `/etc/certs` and `/var/lib/istio/envoy` to the Istio proxy:

    ```bash
    sudo mkdir -p /etc/istio/proxy
    sudo chown -R istio-proxy /var/lib/istio /etc/certs /etc/istio/proxy /etc/istio/config /var/run/secrets /etc/certs/root-cert.pem
    ```

With all files in place, we can start Istio on the virtual machine:

```bash
sudo systemctl start istio
```

You can check that the `istio` service is running with `systemctl status istio`.

At this point, the virtual machine is configured to talk with the Istio's control plane in the Kubernetes cluster.

## Access services from the virtual machine

Let's deploy a Hello world application to the Kubernetes cluster. First, we need to enable the automatic sidecar injection in the `default` namespace:

```
$ kubectl label namespace default istio-injection=enabled
namespace/default labeled
```

Next, create the Hello world Deployment and Service.

```yaml
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

Save the above file to `hello-world.yaml` and deploy it using `kubectl apply -f hello-world.yaml`.

Wait for the Pods to become ready and then go back to the virtual machine and try to access the Kubernetes service:

```
$ curl http://hello-world.default
Hello World
```

You can access any service running within your Kubernetes cluster from the virtual machine.

## Run services on the virtual machine

We can also run a workload on the virtual machine. Switch to the instance and run a simple Python HTTP server:

```bash
$ sudo python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

If you try to curl to the instance IP directly, you will get back a response (directory listing): 

```
$ curl [INSTANCE_IP]
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dt
d">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
...
```

But what we want to do is to add the workload (Python HTTP service) to the mesh. For that reason, we created the VM namespace earlier. So let's create a Kubernetes service that represents the VM workload. Note that the name and the label values equal to the value of the `VM_APP` environment variable we set earlier. Don't forget to deploy the service to the `VM_NAMESPACE`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-vm
  labels:
    app: hello-vm
spec:
  ports:
  - port: 80
    name: http-vm
    targetPort: 80
  selector:
    app: hello-vm
```

Save the above file to `hello-vm-service.yaml` and deploy it to the VM namespace using `kubectl apply -f hello-vm-service.yaml -n vm-namespace`.

Because we used the VM auto-registration, Istio automatically created the WorkloadEntry resource for us. You can check the WorkloadEntry resource with:

```
$ kubectl get workloadentry -n vm-namespace
NAME                  AGE   ADDRESS
hello-vm-10.128.0.7   12m   10.128.0.7
```

We can now use the Kubernetes service name `hello-vm.vm-namespace` to access the workload on the virtual machine. Let's run a Pod inside the cluster and try to access the service from there:

```
$ kubectl run curl --image=radial/busyboxplus:curl -i --tty --rm
If you don't see a command prompt, try pressing enter.
[ root@curl:/ ]$
```

After you get the command prompt in the Pod, you can run curl and access the workload. You should see a directory listing response. Similarly, you will notice a log entry on the instance where the HTTP server is running:

```bash
[ root@curl:/ ]$ curl hello-vm.vm-namespace
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dt
d">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
...
```
