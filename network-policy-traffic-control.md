# 2 Network Policy Traffic Control 

## 2.1 Introduction

Your AKS cluster was deployed with Azure network policies enabled. The Network policies can be used to control traffic between resources in Kubernetes.

In this section you will create two workloads in their own namespaces that initially will be allowed to communicate. After that you will restrict communication between pods.

## 2.2 Create namespaces and workloads

First create a namespace called nginx and an nginx web server

````
kubectl create namespace nginx 
kubectl run nginx --image=nginx --namespace nginx 
````

Next create a name space called busybox and busybox pod, that we will use to send requests to the nginx pod

````
kubectl create namespace busybox 
kubectl run busybox --image=busybox --namespace busybox --namespace busybox -- sleep 3600
````

## 2.3 Generate traffic between pods

Now, *exec* into the busybox container and issue a HTTP GET request towards its IP address

Find the nginx IP address
````
kubectl get pods --namespace nginx -o wide
````

````
kubectl exec -ti busybox --namespace busybox -- sh
````
Then send some traffic
````
wget http://<nginx IP address>
````

And you should get a response similar to this

````
Connecting to 10.224.0.25 (10.224.0.25:80)
saving to 'index.html'
index.html           100% |******************************************************************************************|   615  0:00:00 ETA
'index.html' saved
````

This works because as a default all pods in a Kubernetes cluster can communicate with all other pods in that cluster.

Now delete index.html and exit out of the pod.
````
rm index.html
exit
```` 

## 2.4 Apply "deny" network policy

To restrict pod communication, we can create network policies. 


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

````matchLabels: {}```` means the policy will not match any pod label, which means that no pod will be allowed to communicate with any pod in the ````nginx```` namespace.

If you repeat the wget command from inside the busybox pod, you should find that the request now times out
````
kubectl exec -ti busybox --namespace busybox -- sh

````
and 
````
wget http://<nginx IP address>
````

You will see something similar to this
````
Connecting to 10.224.0.25 (10.224.0.25:80)
````

It times out because we have restricted all communication to the nginx namespace. Use ````ctrl-c```` to break out.

Exit out of the pod.

````
exit
````

## 2.5 Apply "allow" policy

Now apply a new policy that allows traffic into the nginx pod from pods that have the label ````run: busybox```` which are located in the namespace ````busybox````. 

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
      run: nginx
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: busybox 
      podSelector:
        matchLabels:
          run: busybox
EOF
````

This time traffic should be allowed, when you repeat the wget request from the busybox pod.

````
kubectl exec -ti busybox --namespace busybox -- sh
````

And once again:
````
wget http://<nginx IP address>
````

Connecting to 10.224.0.25 (10.224.0.25:80)
saving to 'index.html'
index.html           100% |******************************************************************************************|   615  0:00:00 ETA
'index.html' saved
````

Exit out of the pod.

````
exit
````

## 2.6 Final validation

As a final validation, create a new busybox pod in another namespace and try to connect to nginx


````
kubectl create namespace busybox2 
kubectl run busybox --image=busybox --namespace busybox2 --namespace busybox2 -- sleep 3600
````


Now, *exec* into the new ````busybox2```` pod and issue an HTTP GET request towards the nginx IP address


````
kubectl exec -ti busybox --namespace busybox2 -- sh
````

And one final wget

````
wget http://<nginx IP address>
````

This request will time out, showing that the network policy does not allow traffic from the ````busybox2```` namespace even though the name of the pod is the same as the pod in namespace ````busybox````.

Exit out of the pod.

````
exit
````
