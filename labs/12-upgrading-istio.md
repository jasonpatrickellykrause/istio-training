# Upgrading Istio

In this lab, we will be upgrading Istio 1.8.2 to Istio 1.9.2. To go through the upgrade process, we will start with an empty Kubernetes cluster. You can use the following command to remove any Istio versions you might have installed on the cluster:

```sh
getmesh istioctl x uninstall --purge
```

## Installing Istio 1.8.2

We will use GetMesh to download, switch, and install Istio 1.8.2. First, we need to fetch version 1.8.2:

```sh
$ getmesh fetch --version 1.8.3
fallback to the tetrate flavor since --flavor flag is not given or not supported
fallback to the flavor 0 version which is the latest one in 1.8.3-tetrate

Downloading 1.8.3-tetrate-v0 from https://istio.tetratelabs.io/getmesh/files/istio-1.8.3-tetrate-v0-linux-amd64.tar.gz ...

Istio 1.8.3 Download Complete!

Istio has been successfully downloaded into your system.

For more information about 1.8.3-tetrate-v0, please refer to the release notes:
- https://istio.io/latest/news/releases/1.8.x/announcing-1.8.3/

istioctl switched to 1.8.3-tetrate-v0 now
```

Once GetMesh downloads the specific Istio version, it will automatically switch to it. You can double-check that by running the version command:

```sh
$ getmesh istioctl version
no running Istio pods in "istio-system"
1.8.3-tetrate-v0
```

Make sure the Istio CLI version you're using is 1.8.3.

Let's install Istio 1.8.3 now:

```sh
getmesh istioctl install --set profile=demo --set revision=1-8-3
```

Note that currently, there is a bug in creating the ValidatingWebhookConfiguration if we use the revision label for the initial mesh installation.  For Istio resource validation to work, we need to update the `istiod` service to point to the control plane revision that should handle the validation. Because we used the revision, a service called `istiod-1-8-3` was created - the service should have been called `istiod`.

Let's create this service as a workaround - the actual fix for this issue will be available in Istio 1.10:

```sh
kubectl get service -n istio-system -o json istiod-1-8-3 | jq '.metadata.name = "istiod" | del(.spec.clusterIP, .spec.clusterIPs, .metadata.labels."istio.io/rev")' | kubectl apply -f -
```

We'll deploy a sample application, but before we do that, let's label the `default` namespace with the revision label, so Istio can automatically inject the proxy:

```sh
kubectl label ns default istio.io/rev=1-8-3
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

## Installing Istio 1.9.5

We will now install Istio 1.9.5 with a similar command we used before. However, this time we will set the `revision` flag to indicate the new version.

Let's switch the Istio CLI to the latest version using the following command:

```sh
getmesh fetch --version 1.9.5
```

Next, we can install Istio 1.9.5 and set the revision flag to 1-9-5:

```sh
getmesh istioctl install --set profile=demo --set revision=1-9-5
```

This will create a new control plane (`istiod-1-9-5`), service (`istiod-1-9-5`) and a new sidecar injector. You can see the new control plane components by running the following command:

```sh
$ kubectl get po,svc,mutatingwebhookconfigurations -n istio-system
NAME                                       READY   STATUS    RESTARTS   AGE
pod/istio-egressgateway-6b55554bd8-v5gk7   1/1     Running   0          7m
pod/istio-ingressgateway-9645c75fd-lm6hf   1/1     Running   0          26s
pod/istiod-1-8-3-9c46c5f6f-sv9l8           1/1     Running   0          7m17s
pod/istiod-1-9-5-57fd7478d8-cnd7n          1/1     Running   0          35s

NAME                           TYPE           CLUSTER-IP     EXTERNAL-IP       PORT(S)                                                                                      AGE
service/istio-egressgateway    ClusterIP      10.96.14.173   <none>            80/TCP,443/TCP,15443/TCP                                                                     6m59s
service/istio-ingressgateway   LoadBalancer   10.96.8.83     104.196.241.137   15021:32678/TCP,80:31777/TCP,443:32524/TCP,31400:32358/TCP,15443:32069/TCP,15012:31485/TCP   6m59s
service/istiod-1-8-3           ClusterIP      10.96.4.177    <none>            15010/TCP,15012/TCP,443/TCP,15014/TCP                                                        7m17s
service/istiod-1-9-5           ClusterIP      10.96.2.6      <none>            15010/TCP,15012/TCP,443/TCP,15014/TCP                                                        35s

NAME                                                                                                                WEBHOOKS   AGE
mutatingwebhookconfiguration.admissionregistration.k8s.io/istio-sidecar-injector-1-8-3                              1          7m17s
mutatingwebhookconfiguration.admissionregistration.k8s.io/istio-sidecar-injector-1-9-5                              1          35s
mutatingwebhookconfiguration.admissionregistration.k8s.io/neg-annotation.config.common-webhooks.networking.gke.io   1          3h1m
mutatingwebhookconfiguration.admissionregistration.k8s.io/pod-ready.config.common-webhooks.networking.gke.io        1          3h1m
```

Notice the items in the above output have the `1-9-5` suffix - these are the items that are part of the new control plane we deployed.

Let's check the proxy version of the frontend and customers pods - they should still use 1.8.2:

```sh
$ getmesh istioctl proxy-status | grep $(kubectl get pod -l app=web-frontend -o jsonpath='{.items..metadata.name}') | awk '{print $7}'
1.8.2-tetrate-v0
```

The output should be `1.8.2-tetrate-v0`. On the other the gateways (ingress and egress) are automatically upgraded in-place to 1.9.5 version:

```sh
$ getmesh istioctl proxy-status | grep $(kubectl -n istio-system get pod -l app=istio-ingressgateway -o jsonpath='{.items..metadata.name}') | awk '{print $7}'
1.9.5-tetrate-v0
```

## Upgrade the control plane

The first thing we will switch over to the new version is the ValidatingWebhookConfiguration. At the moment, this points to the existing (old) istiod version (specifically, it points to the `istiod` service we created). We need to update the labels on the `istiod` service to point to revision 1-9-5.

Let's edit the `istiod` service:

```sh
kubectl edit service istiod -n istio-system
```

In the editor under the `selector` field, update the `istio.io/rev` label value to `1-9-5`. Save and close the editor.

We can now delete the revision 1-8-3 of the mutating webhook configuration:

```sh
kubectl delete mutatingwebhookconfiguration istio-sidecar-injector-1-8-3
```

## Update the data plane

The next step is to update the existing Envoy proxies to run the latest Envoy proxy version and connect to the latest Istio control plane.

To upgrade the proxies, we need to configure them to point to the new istiod control plane. The control plane proxies point to is controlled during the sidecar injection and based on the `istio.io/rev` namespace. At the moment, the default namespace has the 1-8-3 label. So for any new deployments to use the latest control plane, we need to change that label to point to revision 1-9-5.

Let's do that now and overwrite the existing label:

```sh
kubectl label ns default istio.io/rev=1-9-5 --overwrite
```

If we created another deployment in the default namespace, the proxies in the new pods would point to the new control plane.

If we want to update the existing deployments, we can restart the Pods to trigger the re-injection and have the new proxies be injected. We can use the rollout and restart command to restart all deployments:

```sh
kubectl rollout restart deployment
```

Once the Pods come back up, the proxies will be configured to talk to `istiod-1-9-5` control plane. We can run the proxy status command to check the proxy versions:

```shell
$ getmesh istioctl proxy-status | grep $(kubectl get pod -l app=web-frontend -o jsonpath='{.items..metadata.name}') | awk '{print $7}'
1.9.5-tetrate-v0
```

Notice that the proxy version is set to the latest Istio version that we deployed. Another way to check this is to list the pods with their labels and check which ones have the `istio.io/rev=1-9-5` label set:

```sh
$ kubectl get po --show-labels
NAME                            READY   STATUS    RESTARTS   AGE    LABELS
customers-v1-5468b9c748-2gn59   2/2     Running   0          2m3s   app=customers,istio.io/rev=1-9-5,pod-template-hash=5468b9c748,security.istio.io/tlsMode=istio,service.istio.io/canonical-name=customers,service.istio.io/canonical-revision=v1,version=v1
web-frontend-5568545fd9-6bpkq   2/2     Running   0          2m3s   app=web-frontend,istio.io/rev=1-9-5,pod-template-hash=5568545fd9,security.istio.io/tlsMode=istio,service.istio.io/canonical-name=web-frontend,service.istio.io/canonical-revision=v1,version=v1
```

## Remove the old control plane

After upgrading both the control plane and the data plane, we can remove the old data plane. The way to remove the old data plane is to uninstall it using its original installation options or, since we labeled our deployments with a revision label, use the revision label.

Let's switch the Istio CLI back to 1.8.2 and remove the old control plane:

```sh
$ getmesh switch --version 1.8.2
istioctl switched to 1.8.2-tetrate-v0 now

$ getmesh istioctl x uninstall --revision=1-8-3
```
