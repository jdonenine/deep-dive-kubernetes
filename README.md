# Deep Dive: Kubernetes - An introduction to Kubernetes

A small companion repo of example commands/configuration files to the discussion topics in these [slides](https://docs.google.com/presentation/d/13mcBvxM4azH3fgWAH0OtBqwaw3CJ0adKQwGg0_VtRYA/edit?usp=sharing).

## Prerequisites

1. [Docker](https://www.docker.com/products/docker-desktop/)
2. [Kind](https://kind.sigs.k8s.io/)
4. [Kubectl](https://kubernetes.io/docs/tasks/tools/)
5. [Kubectx](https://github.com/ahmetb/kubectx/)

## Example 1: Simple Web Server Deployment

In this example we'll be manually deploying a very simple web server deployment.  We'll be using the [nginx](https://www.nginx.com/) as the web server and also as our [ingress controller](https://docs.nginx.com/nginx-ingress-controller/).

Create a local development cluster:

```
kind create cluster --config example-1/kind.config.yaml
```

Confirm that we have the correct context active:

```
kubectx kind-example-1
```

Inspect the nodes that are witin the cluster:

```
kubectl get nodes -o wide
```

Inspect the pods that are running within the cluster's default namespace:

```
kubectl get pods
```

Inspect the pods that are running across all namespaces in the cluster:

```
kubectl get pods --all-namespaces -o wide
```

Install the nginx ingress controller:

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

Check for the ingress-nginx-controller pod to be ready:

```
$ kubectl get pods -n ingress-nginx | grep "ingress-nginx-controller" 
ingress-nginx-controller-86b6d5756c-wmnjs   1/1     Running     0          5m48s
```

Create a namespace to install our application resources into:

```
kubectl create namespace nginx
```

Deploy our resources:

```
kubectl apply -f example-1/deploy.yaml
```

This will deploy the following to the `nginx` namespace:

* A deployment called `nginx` with three pods running the `nginx` image
* A service called `nginx` targeting the `nginx` deployment pods
* An ingress rule called `nginx` that routes all traffic on `/` to the `nginx` service

```
$ kubectl get all -n nginx -o wide
NAME                        READY   STATUS    RESTARTS   AGE     IP            NODE               NOMINATED NODE   READINESS GATES
pod/nginx-8f458dc5b-5dd6d   1/1     Running   0          3m52s   10.244.1.11   example-1-worker   <none>           <none>
pod/nginx-8f458dc5b-cmtf9   1/1     Running   0          3m52s   10.244.1.10   example-1-worker   <none>           <none>
pod/nginx-8f458dc5b-sqdkm   1/1     Running   0          3m52s   10.244.1.12   example-1-worker   <none>           <none>

NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE     SELECTOR
service/nginx   ClusterIP   10.96.149.79   <none>        80/TCP    3m52s   app=nginx

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES   SELECTOR
deployment.apps/nginx   3/3     3            3           3m52s   nginx        nginx    app=nginx

NAME                              DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES   SELECTOR
replicaset.apps/nginx-8f458dc5b   3         3         3       3m52s   nginx        nginx    app=nginx,pod-template-hash=8f458dc5b
```

Launch a browser at [http://localhost](http://localhost) to view the application -- it's just the default nginx home page :-)

You can observe that as the browser is refreshed repeatedly, those requets are served by different pods by observing the logging within each pod, for example:

```
$ kubectl -n nginx get pods
NAME                    READY   STATUS    RESTARTS   AGE
nginx-8f458dc5b-2wr5q   1/1     Running   0          3m44s
nginx-8f458dc5b-4flbr   1/1     Running   0          3m44s
nginx-8f458dc5b-q6dw4   1/1     Running   0          3m44s
```

```
$ kubectl -n nginx logs nginx-8f458dc5b-2wr5q
...
10.244.0.2 - - [17/Aug/2022:14:16:56 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/104.0.0.0 Safari/537.36" "172.19.0.1"
```

```
kubectl -n nginx logs nginx-8f458dc5b-4flbr
...
10.244.0.2 - - [17/Aug/2022:14:18:48 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/104.0.0.0 Safari/537.36" "172.19.0.1"
```

```
$ kubectl -n nginx logs nginx-8f458dc5b-q6dw4
...
10.244.0.2 - - [17/Aug/2022:14:14:51 +0000] "GET / HTTP/1.1" 200 615 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/104.0.0.0 Safari/537.36" "172.19.0.1"
```

You can delete the cluster when done with:

```
kind delete cluster --name example-1
```
