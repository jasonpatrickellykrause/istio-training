# Lab: Installing Istio

In this lab, we will install Istio on your Kubernetes cluster using the Istio operator.

## Prerequisites

To install Istio, we will need a running instance of a Kubernetes cluster. All cloud providers have a managed Kubernetes cluster offering we can use to install Istio service mesh.

We can also run a Kubernetes cluster locally on your computer using one of the following platforms:

- [Minikube](https://istio.io/latest/docs/setup/platform-setup/minikube/)
- [Docker Desktop](https://istio.io/latest/docs/setup/platform-setup/docker/)
- [kind](https://istio.io/latest/docs/setup/platform-setup/kind/)
- [MicroK8s](https://istio.io/latest/docs/setup/platform-setup/microk8s/)

When using a local Kubernetes cluster, make sure your computer meets the minimum requirements for Istio installation (e.g. 16384 MB RAM and 4 CPUs). Also, ensure the Kubernetes cluster version is v1.19.0 or higher.

The section that follow explain how to use Minikube or Kubernetes cluster on Google Cloud Platform.

## Using Minikube 

You can use Minikube with a Hypervisor. Hypervisor choice will depend on your operating system. To install Minikube and the Hypervisor, you can follow the [installation instructions](https://kubernetes.io/docs/tasks/tools/install-minikube/). 

Once we have installed Minikube, we can create and launch the Kubernetes cluster. The below command starts a Minikube cluster using VirtualBox hypervisor.

```bash
minikube start --memory=16384 --cpus=4 --driver=virtualbox
```

Make sure to replace the `--driver=virtualbox` with the name of the Hypervisor you're using. See the table below for available options.

| Flag name | More information |
| --- | --- |
| `hyperkit` | [HyperKit](https://github.com/moby/hyperkit) |
| `hyperv` | [Hyper-V](https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/) | 
| `kvm2` | [KVM](https://www.linux-kvm.org/page/Main_Page) |
|`docker` | [Docker](https://hub.docker.com/search?q=&type=edition&offering=community&sort=updated_at&order=desc) |
| `podman` | [Podman](https://podman.io/getting-started/installation.html)
| `parallels` | [Parallels](https://www.parallels.com/) |
| `virtualbox` | [VirtualBox](https://www.virtualbox.org/) | 
| `vmware` | [VMware Fusion](https://www.vmware.com/products/fusion.html) | 

To check if the cluster is running, we can use the Kubernetes CLI and run the `kubectl get nodes` command:

```bash
$ kubectl get nodes
NAME       STATUS   ROLES    AGE    VERSION
minikube   Ready    master   151m   v1.19.0
```

>Note: if you installed Minikube using [Brew package manager](https://brew.sh), you also have Kubernetes CLI installed.

### Kubernetes CLI

If you need to install the Kubernetes CLI, follow [these instructions](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

We can run `kubectl version` to check if the CLI is installed. You should see the output similar to this one:

```bash
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.2", GitCommit:"8b5a19147530eaac9476b0ab82980b4088bbc1b2", GitTreeState:"clean", BuildDate:"2021-09-15T21:38:50Z", GoVersion:"go1.16.8", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"20+", GitVersion:"v1.20.10-gke.1600", GitCommit:"ef8e9f64449d73f9824ff5838cea80e21ec6c127", GitTreeState:"clean", BuildDate:"2021-09-06T09:24:20Z", GoVersion:"go1.15.15b5", Compiler:"gc", Platform:"linux/amd64"}
```

## Using Google Cloud Platform

The instructor will give you account information you can use to log into [Google Cloud Platform](https://cloud.google.com/).

After you've logged in click the Organization and Project dropdown on the top left of the screen as shown in the figure:

![Select project](./img/1-no-organization.png)

From the organization/project window select the organization (there should be only one) and then click the project name that shows up in the list of projects (there should also be only one project in the list).

Note that you might have to click the **ALL** tab to see the list of all projects and select the project from there.

![Select org and project](./img/1-project-org-select.png)

Once you've selected the project and logged in, click the **Activate Cloud Shell** button on the top-right of the screen as shown in the figure below:

![Activate Cloud Shell](./img/1-activate-cloudshell.png)

This is the terminal you will use to go through the labs.

### Connecting to the Kubernetes cluster

A Kubernetes cluster was provisioned for you. In order to connect to it from the Cloud Shell, you need to run a GCloud CLI command.

1. Click the hamburger button on the top left side of the screen.
1. From the Compute section, select **Kubernetes Engine** and click **Clusters**.

  ![Cluster list](./img/1-cluster-list.png)

1. From the context menu (click the three dots), click the **Connect** button.
1. Under the Command-line access, click the **Copy** button to copy the gcloud command to your clipboard.

  ![Command access](./img/1-cmd-access.png)

1. Paste the command to your cloud shell terminal to connect to the cluster.

You will also be prompted to authorize cloud shell - you can safely click the Authorize button.

![Authorize cloud Shell](./img/1-authorize-cloud-shell.png)

To check if you're successfully connected to the cluster, you can run `kubectl get nodes` and you should see the output similar to this one:

```sh
user@cloudshell:~$ kubectl get nodes
NAME                                                 STATUS   ROLES    AGE   VERSION
project-name-default-pool-e46ed8ba-0sqm   Ready    <none>   23m   v1.18.17-gke.100
project-name-default-pool-e46ed8ba-vkdk   Ready    <none>   23m   v1.18.17-gke.100
project-name-default-pool-e46ed8ba-xr0n   Ready    <none>   23m   v1.18.17-gke.100
user@cloudshell:~$
```

<!-- ## Install GetMesh CLI

The first step is to download GetMesh CLI. You can install GetMesh on macOS and Linux platforms. We can use the following command to download the latest version of GetMesh and certified Istio:

```sh
curl -sL https://istio.tetratelabs.io/getmesh/install.sh | bash
```

After installation completes, open a new tab terminal (click the + button in the top bar of the cloud shell).

We can now run the version command to ensure GetMesh is successfully installed. For example:

```sh
$ getmesh version
getmesh version: 1.1.3
active istioctl: 1.11.3-tetrate-v0
no running Istio pods in "istio-system"
1.11.3-tetrate-v0
```

The version command outputs the version of GetMesh, the version of active Istio CLI, and versions of Istio installed on the Kubernetes cluster. -->

## Download and install Istio

<!-- GetMesh communicates with the active Kubernetes cluster from the Kubernetes config file. Make sure you have the correct Kubernetes context selected (`kubectl config get-contexts`) before installing Istio. -->

The recommended profile for production deployments is the `default` profile. We will be installing the `demo` profile as it contains all core components, has a high level of tracing and logging enabled, and is meant for learning about different Istio features.

We can also start with the `minimal` component and individually install other features, like ingress and egress gateway, later.

To install the demo profile of Istio on a currently active Kubernetes cluster, we can have to download Istio first:

```sh
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.12.1 sh - 
```

Next, let's copy the Istio CLI to the `/usr/local/bin` folder:

```sh
sudo cp istio-1.12.1/bin/istioctl /usr/local/bin
```

>Google Cloud Shell comes preinstalled with an older Istio CLI version.

We can now install the demo profile of Istio:

```sh
istioctl install --set profile=demo -y
```
<!-- 
```sh
getmesh istioctl install --set profile=demo
```

GetMesh will check the cluster to make sure it is ready for Istio installation. Make sure you confirm the installation by when prompted by typing "y" and GetMesh will proceed with installation.

Once the installation completes you will see the following line in the output:

```sh
✔ Istio is installed and verified successfully
``` 

We can run the version comand again (`getmesh version`). You’ll notice that the output shows the control plane and data plane versions installed on the cluster.

```sh
$ getmesh version
getmesh version: 1.1.3
active istioctl: 1.11.3-tetrate-v0
client version: 1.11.3-tetrate-v0
control plane version: 1.11.3-tetrate-v0
data plane version: 1.11.3-tetrate-v0 (2 proxies)
```
-->

To check the status of the installation, we can look at the status of the Pods in the `istio-system` namespace:

```bash
$ kubectl get po -n istio-system
NAME                                    READY   STATUS    RESTARTS   AGE
istio-egressgateway-6db9994577-sn95p    1/1     Running   0          79s
istio-ingressgateway-58649bfdf4-cs4fk   1/1     Running   0          79s
istiod-dd4b7db5-nxrjv                   1/1     Running   0          111s
```

The operator has finished installing Istio when all Pods are running.

## Enable sidecar injection 

As we've learned in the previous section, service mesh needs the sidecar proxies running alongside each application.

To inject the sidecar proxy into an existing Kubernetes deployment, we can use `kube-inject` action in the `istioctl` command.

However, we can also enable automatic sidecar injection on any Kubernetes namespace. If we label the namespace with `istio-injection=enabled`, Istio automatically injects the sidecars for any Kubernetes Pods we create in that namespace. 

Let's enable automatic sidecar injection on the `default` namespace by adding a label:

```bash
$ kubectl label namespace default istio-injection=enabled
namespace/default labeled
```

To check the namespace is labeled, run the command below. The `default` namespace should be the only one with the value `enabled`.

```bash
$ kubectl get namespace -L istio-injection
NAME              STATUS   AGE     ISTIO-INJECTION
default           Active   3m49s   enabled
istio-system      Active   82s
kube-node-lease   Active   3m51s
kube-public       Active   3m51s
kube-system       Active   3m51s
```

We can now try creating a Deployment in the `default` namespace and observe the injected proxy. We will create a deployment called `my-nginx` with a single container using image `nginx`:

```
$ kubectl create deploy my-nginx --image=nginx
deployment.apps/my-nginx created
```

If we look at the Pods, you will notice there are two containers in the Pod:

```
$ kubectl get po
NAME                        READY   STATUS    RESTARTS   AGE
my-nginx-6b74b79f57-ks7p8   2/2     Running   0          62s
``` 

Similarly, describing the Pod shows Kubernetes created both an `nginx` container and an `istio-proxy` container:

```
$ kubectl describe po my-nginx-6b74b79f57-ks7p8 
...
Events:
  Type     Reason       Age   From               Message
  ----     ------       ----  ----               -------
  Normal   Scheduled    30s   default-scheduler  Successfully assigned default/my-nginx-6b74b79f57-ks7p8 to gke-cluster-1-default-pool-ab26b687-9nsr
  Normal   Pulled       27s   kubelet            Container image "containers.istio.tetratelabs.com/proxyv2:1.12.1" already present on machine
  Normal   Created      27s   kubelet            Created container istio-init
  Normal   Started      27s   kubelet            Started container istio-init
  Normal   Pulling      27s   kubelet            Pulling image "nginx"
  Normal   Pulled       23s   kubelet            Successfully pulled image "nginx" in 3.89712405s
  Normal   Created      22s   kubelet            Created container nginx
  Normal   Started      22s   kubelet            Started container nginx
  Normal   Pulled       22s   kubelet            Container image "containers.istio.tetratelabs.com/proxyv2:1.12.1" already present on machine
  Normal   Created      22s   kubelet            Created container istio-proxy
  Normal   Started      22s   kubelet            Started container istio-proxy
```

To remove the deployment, run the delete command:

```bash
$ kubectl delete deployment my-nginx
deployment.apps "my-nginx" deleted
```

### Uninstalling Istio

To completely uninstall Istio from the cluster, run the following command:

```sh
istioctl x uninstall --purge
```

Press `y` to proceed with uninstalling Istio.

## Installing Istio using IstioOperator

To get started we'll initialize the default Istio operator first. The init command deploys the operator to the `istio-operator` image and it configures it to watch the `istio-system` namespace. That means we'll have to create the IstioOperator resouce in the `istio-system` namespace so it gets picked up by the operator.

Let's initalize the operator first:

```sh
istioctl operator init
```

Istio operator is deployed to the `istio-operator` namespace:

```sh
$ kubectl get po -n istio-operator
NAME                              READY   STATUS    RESTARTS   AGE
istio-operator-54958d5898-mkrpf   1/1     Running   0          99s
```

>Note: If using earlier versions of Istio (e.g. Istio 1.9), you'll have to manually create the `istio-system` namespace. With latest versions of Istio, namespace is created automatically when operator is initialized. 

Next, we can create the IstioOperator resource to install Istio. We'll use the demo profile - that profile includes the `istiod`, an `istio-ingressgateway` and an egress gateway:

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: demo-installation
spec:
  profile: demo
```

Save the above to `demo-installation.yaml` and create the resource with `kubectl apply -f demo-installation.yaml`.

We can check the status of the installation by listing the Istio operator resource. The installation is completed once the STATUS column shows HEALTHY:

```sh
$ kubectl get iop -A
NAMESPACE      NAME                   REVISION   STATUS        AGE
istio-system   demo-installation              HEALTHY       67s
```

>Note: you can also look at the more detailed installation logs from the Istio operator pod.

### Updating the operator

To update the operator we can use kubectl and apply the updated IstioOperator resource. For example, if we wanted to remove the egress gateway we could update the IstioOperator resource like this:

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
      enabled: false
```

Save the above YAML to `iop-egress.yaml` and apply it to the cluster using `kubectl apply -f iop-egress.yaml`.

If we list the IstioOperator resource you'll notice the status has changed to `RECONCILING` and once the egress is removed, the status changes back to HEALTHY. We can also look at the pods in the `istio-system` namespace to see that the egress gateway pod was removed.

Another option for updating the Istio installation is to create separate IstioOperator resources. That way, you can have a resource for the base installation and separately apply different operators using an empty installation profile.

For example, here's how you could create a separate IstioOperator resource that only deploys an internal ingress gateway:

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: internal-gateway-only
  namespace: istio-system
spec:
  profile: empty
  components:
    ingressGateways:
      - namespace: some-namespace
        name: ilb-gateway
        enabled: true
        label:
          istio: ilb-gateway
        k8s:
          serviceAnnotations:
            networking.gke.io/load-balancer-type: "Internal"
```

## Cleanup 

Note that you'll be using the same cluster throughout the workshop, so you don't have to delete Istio. However, if you want to try it out, you can always delete and re-install it again.

To delete Istio we have to delete the IstioOperator resource:

```sh
kubectl delete iop demo-installation -n istio-system
```

Once Istio is deleted, you have to also remove the IstioOperator:

```sh
istioctl operator remove
```

Finally, remove the `istio-system` namespace: `kubectl delete ns istio-system`.
