# Deploying the DNS Cluster Add-on

In this lab you will deploy the [DNS add-on](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) which provides DNS based service discovery to applications running inside the Kubernetes cluster.

## The DNS Cluster Add-on

Deploy the `kube-dns` cluster add-on:

```bash
kubectl create -f https://storage.googleapis.com/kubernetes-the-hard-way/kube-dns.yaml
```

> output

```console
service "kube-dns" created
serviceaccount "kube-dns" created
configmap "kube-dns" created
deployment.extensions "kube-dns" created
```

List the pods created by the `kube-dns` deployment:

```bash
kubectl get pods -l k8s-app=kube-dns -n kube-system
```

> output

```console
NAME                        READY     STATUS    RESTARTS   AGE
kube-dns-3097350089-gq015   3/3       Running   0          20s
```

## Verification

Create a `busybox` deployment:

```bash
kubectl run busybox --image=busybox --command -- sleep 3600
```

List the pod created by the `busybox` deployment:

```bash
kubectl get pods -l run=busybox
```

> output

```console
NAME                       READY     STATUS    RESTARTS   AGE
busybox-2125412808-mt2vb   1/1       Running   0          15s
```

Retrieve the full name of the `busybox` pod:

```bash
POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")
```

Execute a DNS lookup for the `kubernetes` service inside the `busybox` pod:

```bash
kubectl exec -ti $POD_NAME -- nslookup kubernetes
```

> output

```console
Server:    10.32.0.10
Address 1: 10.32.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.32.0.1 kubernetes.default.svc.cluster.local
```

Next: [Smoke Test](13-smoke-test.md)
