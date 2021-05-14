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

In this training you will be using a Kubernetes instance running on Google Cloud Platform.

The instructor will give you account information you can use to log into [Google Cloud Platform](https://cloud.google.com/).

After you've logged in click the Organization and Project dropdown on the top left of the screen as shown in the figure:

![Select project](./img/1-no-organization.png)

From the organization/project window select the organization (there should be only one) and then click the project name that shows up in the list of projects (there should also be only one project in the list).

![Select org and project](./img/1-project-org-select.png)


Once you've selected the project and logged in, click the **Activate Cloud Shell** button on the top-right of the screen as shown in the figure below:

![Activate Cloud Shell](./img/1-activate-cloudshell.png)

This is the terminal you will use to go through the labs. You will also be prompted to authorize cloud shell - you can safely click the Authorize button.

![Authorize cloud Shell](./img/1-authorize-cloud-shell.png)

### Connecting to the Kubernetes cluster

A Kubernetes cluster was provisioned for you. In order to connect to it from the Cloud Shell, you need to run a GCloud CLI command.

1. Click the hamburger button on the top left side of the screen.
1. From the Compute section, select **Kubernetes Engine** and click **Clusters**.

  ![Cluster list](./img/1-cluster-list.png)

1. From the context menu (click the three dots), click the **Connect** button.
1. Under the Command-line access, click the **Copy** button to copy the gcloud command to your clipboard.

  ![Command access](./img/1-cmd-access.png)

1. Paste the command to your cloud shell terminal to connect to the cluster.

To check if you're successfully connected to the cluster, you can run `kubectl get nodes` and you should see the output similar to this one:

```sh
user@cloudshell:~$ kubectl get nodes
NAME                                                 STATUS   ROLES    AGE   VERSION
project-name-default-pool-e46ed8ba-0sqm   Ready    <none>   23m   v1.18.17-gke.100
project-name-default-pool-e46ed8ba-vkdk   Ready    <none>   23m   v1.18.17-gke.100
project-name-default-pool-e46ed8ba-xr0n   Ready    <none>   23m   v1.18.17-gke.100
user@cloudshell:~$
```

### Install GetIstio CLI

The first step is to download GetIstio CLI. You can install GetIstio on macOS and Linux platforms. We can use the following command to download the latest version of GetIstio and certified Istio:

```sh
curl -sL https://tetrate.bintray.com/getistio/download.sh | bash
```

After installation completes, open a new tab terminal (click the + button in the top bar of the cloud shell).

We can now run the version command to ensure GetIstio is successfully installed. For example:

```sh
$ getistio version
getistio version: 1.0.5
active istioctl: 1.9.1-tetrate-v0
no running Istio pods in "istio-system"
1.9.1-tetrate-v0
```

The version command outputs the version of GetIstio, the version of active Istio CLI, and versions of Istio installed on the Kubernetes cluster.

## Install Istio

GetIstio communicates with the active Kubernetes cluster from the Kubernetes config file. Make sure you have the correct Kubernetes context selected (`kubectl config get-contexts`) before installing Istio.

The recommended profile for production deployments is the `default` profile. We will be installing the `demo` profile as it contains all core components, has a high level of tracing and logging enabled, and is meant for learning about different Istio features.

We can also start with the `minimal` component and individually install other features, like ingress and egress gateway, later.

To install the demo profile of Istio on a currently active Kubernetes cluster, we can use the `getistio istioctl` command like this:

```sh
getistio istioctl install --set profile=demo
```

GetIstio will check the cluster to make sure it is ready for Istio installation. Make sure you confirm the installation by when prompted by typing "y" and GetIstio will proceed with installation.

Once the installation completes you will see the following line in the output:

```sh
✔ Istio is installed and verified successfully
```

We can run the version comand again (`getistio version`). You’ll notice that the output shows the control plane and data plane versions installed on the cluster.

```sh
$ getistio version
getistio version: 1.0.5
active istioctl: 1.9.1-tetrate-v0
client version: 1.9.1-tetrate-v0
control plane version: 1.9.1-tetrate-v0
data plane version: 1.9.1-tetrate-v0 (2 proxies)
```

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
NAME              STATUS   AGE   ISTIO-INJECTION
default           Active   32m   enabled
istio-operator    Active   27m   disabled
istio-system      Active   15m
kube-node-lease   Active   32m
kube-public       Active   32m
kube-system       Active   32m
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
my-nginx-6b74b79f57-hmvj8   2/2     Running   0          62s
``` 

Similarly, describing the Pod shows Kubernetes created both an `nginx` container and an `istio-proxy` container:

```
$ kubectl describe po my-nginx-6b74b79f57-hmvj8
...
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  28s   default-scheduler  Successfully assigned default/my-nginx-9b596c8c4-2v5d7 to gke-fico-may-2021-1-009-default-pool-e46ed8ba-0sqm
  Normal  Pulling    27s   kubelet            Pulling image "tetrate-docker-getistio-docker.bintray.io/proxyv2:1.9.1-tetrate-v0"
  Normal  Pulled     26s   kubelet            Successfully pulled image "tetrate-docker-getistio-docker.bintray.io/proxyv2:1.9.1-tetrate-v0"
  Normal  Created    26s   kubelet            Created container istio-init
  Normal  Started    25s   kubelet            Started container istio-init
  Normal  Pulling    25s   kubelet            Pulling image "nginx"
  Normal  Pulled     20s   kubelet            Successfully pulled image "nginx"
  Normal  Created    19s   kubelet            Created container nginx
  Normal  Started    19s   kubelet            Started container nginx
  Normal  Pulling    19s   kubelet            Pulling image "tetrate-docker-getistio-docker.bintray.io/proxyv2:1.9.1-tetrate-v0"
  Normal  Pulled     17s   kubelet            Successfully pulled image "tetrate-docker-getistio-docker.bintray.io/proxyv2:1.9.1-tetrate-v0"
  Normal  Created    17s   kubelet            Created container istio-proxy
  Normal  Started    17s   kubelet            Started container istio-proxy
```

To remove the deployment, run the delete command:

```bash
$ kubectl delete deployment my-nginx
deployment.apps "my-nginx" deleted
```

### Updating and uninstalling Istio

**Note**
If you uninstall Istio, make sure you install it again, because the remaining labs depend on Istio being installed.

To completely uninstall Istio from the cluster, run the following command:

```sh
getistio istioctl x uninstall --purge
```