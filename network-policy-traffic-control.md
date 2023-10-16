# Network Policy Traffic Control 

## Introduction

Your AKS cluster was deployed with Azure network policies enabled. The Network policies can be used to control traffic between resources in Kubernetes.

In this section you will create two workloads in their own namespaces that initially will be allowed to communicate. After that you will restrict communication between pods.

## Create namespaces and workloads

First create a namespace called nginx and an nginx web server

````
kubectl create namespace nginx 
kubectl create deployment nginx --image=nginx --replicas=1 --namespace nginx
````

Next create a name space called busybox and busybox pod, that we will use to send requests to the nginx pod

````
kubectl create namespace busybox 
kubectl create deployment busybox --image=busybox --replicas=1 --namespace busybox

````

## Generate traffic between pods

Now, *exec* into the busybox container and issue a HTTP GET request towrds its IP address

````
kubectl exec -ti busybox -- bash
curl http://<nginx IP address>
````

And you should get a response similar to this

````
HTTP/1.1 200 OK
Server: nginx/1.19.10
Date: Sun, 15 Oct 2023 15:23:00 GMT
Content-Type: text/html; charset=UTF-8
Content-Length: 13
Connection: keep-alive
...
...

````

This works because as a default all pods in a Kubernetes cluster can communicate with all other pods in that cluster.

Now exit out of the container:
````
exit
````

## Apply network policies

To restrict pod coomunication, we can create network policies. 


This first policy will prevent all traffic to the nginx namespace. 

````
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-ingress-nginx
  namespace: nginx
spec:
  podSelector: {}
  ingress:
  - from:
    - podSelector:
        matchLabels: {}
EOF
````


If you repeat the curl command form inside the busybox pod, you should find that the request now times out
````
kubectl exec -ti busybox -- bash
curl http://<nginx IP address>

````

It times out because we have restricted all communication to the nginx namespace.

Now apply a new policy that allows traffic into the nginx pod from pods that have the label ````app: busybox```` which are located in the namespace ````busybox``````.

````
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-busybox
  namespace: nginx 
spec:
  podSelector:
    matchLabels:
      app: nginx
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: busybox 
      podSelector:
        matchLabels:
          app: busybox
EOF
````

This time traffic should be allowed when you repeat the curl request from the busybox pod.

````
kubectl exec -ti busybox -- bash
curl http://<nginx IP address>

HTTP/1.1 200 OK
Server: nginx/1.19.10
Date: Sun, 15 Oct 2023 15:23:00 GMT
Content-Type: text/html; charset=UTF-8
Content-Length: 13
Connection: keep-alive

...
...
...

````