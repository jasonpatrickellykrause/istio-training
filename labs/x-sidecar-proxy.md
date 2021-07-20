# Exploring the sidecar proxy

In this lab we'll take a look at the iptables rules that are responsible for intercepting inbound and outbound traffic to the application pod.

We'll start with a simple `httpbin` deployment below.

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

Save the above YAML `httpbin.yaml` file. We'll use Istio CLI to manually inject the sidecar to the deployment and create it:

```sh
getmesh istioctl kube-inject -f httpbin.yaml | kubectl apply -f -
```

When the deployment is created we can edit it and configure the `istio-proxy` container to run as a priviledged container. This will allow us to look run the iptables command with root privileges.

Let's open the deployment YAML in the editor using the `kubectl edit deployment httpbin` command. This will open the vim editor with the deployment YAML.

Find the containers section, specifically the `istio-proxy` container and change the following two values in the `securityContext` to `true`:

```yaml
securityContext:
  allowPrivilegeEscalation: true
  privileged: true
```

Save the changes and exit the editor.

We can now use `kubectl exec` to get the terminal inside the `istio-proxy` container and look at the iptables rules. You can get the Pod name by running `kubectl get pods`, then use the following command to get a terminal inside the `istio-proxy` container:

```sh
$ kubectl exec -it [httpbin-pod-name] -c istio-proxy -- /bin/bash
```

Once inside the container we can list all rules in all chains in the NAT table using `sudo iptables -L -t nat`. You'll get an output like the one below:

```sh
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
ISTIO_INBOUND  tcp  --  anywhere             anywhere

Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
ISTIO_OUTPUT  tcp  --  anywhere             anywhere

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination

Chain ISTIO_INBOUND (1 references)
target     prot opt source               destination
RETURN     tcp  --  anywhere             anywhere             tcp dpt:15008
RETURN     tcp  --  anywhere             anywhere             tcp dpt:22
RETURN     tcp  --  anywhere             anywhere             tcp dpt:15090
RETURN     tcp  --  anywhere             anywhere             tcp dpt:15021
RETURN     tcp  --  anywhere             anywhere             tcp dpt:15020
ISTIO_IN_REDIRECT  tcp  --  anywhere             anywhere

Chain ISTIO_IN_REDIRECT (3 references)
target     prot opt source               destination
REDIRECT   tcp  --  anywhere             anywhere             redir ports 15006

Chain ISTIO_OUTPUT (1 references)
target     prot opt source               destination
RETURN     all  --  127.0.0.6            anywhere
ISTIO_IN_REDIRECT  all  --  anywhere            !localhost            owner UID match istio-proxy
RETURN     all  --  anywhere             anywhere             ! owner UID match istio-proxy
RETURN     all  --  anywhere             anywhere             owner UID match istio-proxy
ISTIO_IN_REDIRECT  all  --  anywhere            !localhost            owner GID match istio-proxy
RETURN     all  --  anywhere             anywhere             ! owner GID match istio-proxy
RETURN     all  --  anywhere             anywhere             owner GID match istio-proxy
RETURN     all  --  anywhere             localhost
ISTIO_REDIRECT  all  --  anywhere             anywhere

Chain ISTIO_REDIRECT (1 references)
target     prot opt source               destination
REDIRECT   tcp  --  anywhere             anywhere             redir ports 15001
```

You can also inspect individual chains by specifying the chain name:

```
$ sudo iptables -L ISTIO_INBOUND -t nat
Chain ISTIO_INBOUND (1 references)
target     prot opt source               destination
RETURN     tcp  --  anywhere             anywhere             tcp dpt:15008
RETURN     tcp  --  anywhere             anywhere             tcp dpt:22
RETURN     tcp  --  anywhere             anywhere             tcp dpt:15090
RETURN     tcp  --  anywhere             anywhere             tcp dpt:15021
RETURN     tcp  --  anywhere             anywhere             tcp dpt:15020
ISTIO_IN_REDIRECT  tcp  --  anywhere             anywhere
```

For example, the above output shows the rules in the `ISTIO_INBOUND` chain. If the port matches any of the listed ports (15008, 22, 15090,15021, 15020) the target for those rules is set to RETURN. The return target simply means to stop going through the rules in this chain and return back to the next rule from the previous chain.

If we get through all the rules we end up with the last one which redirects the traffic to `ISTIO_IN_REDIRECT` chain. Let's look at the details of the `ISTIO_IN_REDIRECT`:

```sh
$ sudo iptables -L ISTIO_IN_REDIRECT -t nat
Chain ISTIO_IN_REDIRECT (3 references)
target     prot opt source               destination
REDIRECT   tcp  --  anywhere             anywhere             redir ports 15006
```

There's a single rule in this chain that redirects the traffic to the port 15006 - which is the port for all inbound traffic on the Envoy proxy.

A more complex set of rules are in the `ISTIO_OUTPUT` chain:

```sh
$ sudo iptables -L ISTIO_OUTPUT -t nat
Chain ISTIO_OUTPUT (1 references)
target     prot opt source               destination
RETURN     all  --  127.0.0.6            anywhere
ISTIO_IN_REDIRECT  all  --  anywhere            !localhost            owner UID match istio-proxy
RETURN     all  --  anywhere             anywhere             ! owner UID match istio-proxy
RETURN     all  --  anywhere             anywhere             owner UID match istio-proxy
ISTIO_IN_REDIRECT  all  --  anywhere            !localhost            owner GID match istio-proxy
RETURN     all  --  anywhere             anywhere             ! owner GID match istio-proxy
RETURN     all  --  anywhere             anywhere             owner GID match istio-proxy
RETURN     all  --  anywhere             localhost
ISTIO_REDIRECT  all  --  anywhere             anywhere
```

This chain is involved in the output path of the request (i.e. traffic leaving the proxy/application) - this is also the place where the checks for istio-proxy GID/UID (1337) is being done to prevent redirecting the traffic back to the proxy again.

## Cleanup

Delete the created resources:

```sh
kubectl delete svc httpbin
kubectl delete deploy httpbin
kubectl delete serviceaccount httpbin
```