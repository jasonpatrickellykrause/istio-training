# Upgrading Istio

In this lab, we will be upgrading Istio 1.10.3 to Istio 1.11.3. To go through the upgrade process, we will start with an empty Kubernetes cluster. You can use the following command to remove any Istio versions you might have installed on the cluster:

```sh
getmesh istioctl x uninstall --purge
```

## Installing Istio 1.10.3

We will use GetMesh to download, switch, and install Istio 1.10.3. First, we need to fetch version 1.10.3:

```sh
$ getmesh fetch --version 1.10.3
fallback to the tetrate flavor since --flavor flag is not given or not supported
fallback to the flavor 0 version which is the latest one in 1.10.3-tetrate

Downloading 1.10.3-tetrate-v0 from https://istio.tetratelabs.io/getmesh/files/istio-1.10.3-tetrate-v0-linux-amd64.tar.gz ...

Istio 1.10.3 Download Complete!

Istio has been successfully downloaded into your system.

For more information about 1.10.3-tetrate-v0, please refer to the release notes:
- https://istio.io/latest/news/releases/1.10.x/announcing-1.10.3/

istioctl switched to 1.10.3-tetrate-v0 now
```

Once GetMesh downloads the specific Istio version, it will automatically switch to it. You can double-check that by running the version command:

```sh
$ getmesh istioctl version
no running Istio pods in "istio-system"
1.10.3-tetrate-v0
``` 

Let's install Istio 1.10.3 now:

```sh
$ getmesh istioctl install --set profile=demo --set revision=1-10-3
```

We'll deploy a sample application, but before we do that, let's label the `default` namespace with the revision label, so Istio can automatically inject the proxy:

```sh
kubectl label ns default istio.io/rev=1-10-3
```

We have labeled the `default` namespace with the `istio.io/rev` label because we used a revision label during the installation. 

With Istio installed and namespace labeled, let's deploy a sample application - the web frontend and customer service.

Run `kubectl apply -f upgrade-demo.yaml` (the YAML file is in the `yaml/` folder).

Let's get the gateway's external IP address and store it in the GATEWAY_URL variable:

```sh
export GATEWAY_URL=$(kubectl get service istio-ingressgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

We can now use the [ModHeader](https://chrome.google.com/webstore/detail/modheader/idgpnmonknjnojddfkpgkljpfnnfcklj?hl=en) extension to modify the headers from the browser and check that the applications are running. Alternatively, we can use cURL and add the header to the request like this:

```sh
curl -H "Host: frontend.example.io" $GATEWAY_URL
```

Note that we are adding the host manually to the requests because we configured the Gateway resource to listen on the `frontend.example.io` host.

## Installing Istio 1.11.3

We will now install Istio 1.11.3 with a similar command we used before. However, this time we will set the `revision` flag to indicate the new version.

Let's switch the Istio CLI to the latest version using the following command:

```sh
$ getmesh fetch --version 1.11.3
```

Next, we can install Istio 1.11.3 and set the revision flag to 1-11-3:

```sh
$ getmesh istioctl install --set profile=demo --set revision=1-11-3
```

This will create a new control plane (`istiod-1-11-3`), service (`istiod-1-11-3`) and a new sidecar injector. You can see the new control plane components by running the following command:

```sh
$ kubectl get po,svc,mutatingwebhookconfigurations -n istio-system
NAME                                        READY   STATUS    RESTARTS   AGE
pod/istio-egressgateway-5ffc4d686b-ck97l    1/1     Running   0          38s
pod/istio-ingressgateway-6dddffb46b-6mbcw   1/1     Running   0          38s
pod/istiod-1-10-3-8587dfcf6d-b7kvn          1/1     Running   0          7m31s
pod/istiod-1-11-3-7d649f5b6-mmbdl           1/1     Running   0          43s

NAME                           TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                                                                      AGE
service/istio-egressgateway    ClusterIP      10.64.11.143   <none>           80/TCP,443/TCP                                                               7m16s
service/istio-ingressgateway   LoadBalancer   10.64.11.9     35.233.145.150   15021:30320/TCP,80:31886/TCP,443:31943/TCP,31400:30767/TCP,15443:32113/TCP   7m16s
service/istiod-1-10-3          ClusterIP      10.64.10.221   <none>           15010/TCP,15012/TCP,443/TCP,15014/TCP                                        7m31s
service/istiod-1-11-3          ClusterIP      10.64.2.49     <none>           15010/TCP,15012/TCP,443/TCP,15014/TCP                                        43s

NAME                                                                                                                WEBHOOKS   AGE
mutatingwebhookconfiguration.admissionregistration.k8s.io/istio-sidecar-injector-1-10-3                             2          7m32s
mutatingwebhookconfiguration.admissionregistration.k8s.io/istio-sidecar-injector-1-11-3                             2          43s
mutatingwebhookconfiguration.admissionregistration.k8s.io/neg-annotation.config.common-webhooks.networking.gke.io   1          27m
mutatingwebhookconfiguration.admissionregistration.k8s.io/pod-ready.config.common-webhooks.networking.gke.io        1          27m
```

Notice the items in the above output have the `1-11-3` suffix - these are the items that are part of the new control plane we deployed.

Let's check the proxy version of the frontend and customers pods - they should still use 1.10.3:

```sh
$ getmesh istioctl proxy-status | grep $(kubectl get pod -l app=web-frontend -o jsonpath='{.items..metadata.name}') | awk '{print $7}'
1.10.3-tetrate-v0
```

The output should be `1.10.3-tetrate-v0`. On the other hand the gateways (ingress and egress) are automatically upgraded in-place to 1.11.3 version:

```sh
$ getmesh istioctl proxy-status | grep $(kubectl -n istio-system get pod -l app=istio-ingressgateway -o jsonpath='{.items..metadata.name}') | awk '{print $7}'
1.11.3-tetrate-v0
```

## Update the data plane

The next step is to update the existing Envoy proxies to run the latest Envoy proxy version and connect to the latest Istio control plane.

To upgrade the proxies, we need to configure them to point to the new istiod control plane. The control plane proxies point to is controlled during the sidecar injection and based on the `istio.io/rev` namespace. At the moment, the default namespace has the 1-10-3 label. So for any new deployments to use the latest control plane, we need to change that label to point to revision 1-11-3.

Let's do that now and overwrite the existing label:

```sh
kubectl label ns default istio.io/rev=1-11-3 --overwrite
```

If we created another deployment in the default namespace, the proxies in the new pods would point to the new control plane. 

If we want to update the existing deployments, we can restart the Pods to trigger the re-injection and have the new proxies be injected. We can use the rollout and restart command to restart all deployments:

```sh
kubectl rollout restart deployment
```

Once the Pods come back up, the proxies will be configured to talk to `istiod-1-11-3` control plane. We can run the proxy status command to check the proxy versions:

```
$ getmesh istioctl proxy-status | grep $(kubectl get pod -l app=web-frontend -o jsonpath='{.items..metadata.name}') | awk '{print $7}'
1.11.3-tetrate-v0
```

Notice that the proxy version is set to the latest Istio version that we deployed.

## Remove the old control plane

After upgrading both the control plane and the data plane, we can remove the old data plane. The way to remove the old data plane is to uninstall it using its original installation options or, since we labeled our deployments with a revision label, use the revision label.

Let's switch the Istio CLI back to 1.10.3 and remove the old control plane:

```sh
$ getmesh switch --version 1.10.3
istioctl switched to 1.8.2-tetrate-v0 now

$ getmesh istioctl x uninstall --revision=1-10-3
```