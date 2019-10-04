## Basic up and running
To run, begin with:
`minikube start`

To use these configs, use:
`kubectl apply -f client-node-port.yml`
`kubectl apply -f client-pod.yml`

To check their status, use:
`kubectl get pods`
`kubectl get status`

To find the ip to connect to in your browser, use:
`minikube ip`
Then, you go to the port listed in client-node-port.yml under **nodePort**

To get info about a pod, you can use:
`kubectl describe pod client-pod`

## Pods vs Deployments
When you update a config (for ex: client-pod.yml)
you will need to re-run: `kubectl apply -f client-pod.yml` 
**How does k8s know which pod to update?**
As long as the name and kind are left the same, it knows!
Changing those will screw this up. But, you can only change _some_ configs

**Pods are only used in a development env (can't update most configs)**

Deployments run a set of IDENTICAL pods
Monitors the state of each pod
Good for development AND PRODUCTION
Deployments have _pod templates_ (number of containers, name, port, image)

_matchLabels_ and _labels_ will be the same in most instances. **Why?**. Because we may only want to monitor a specific pod, in which case _matchLabels_ can be told which one to monitor.

## How to delete a pod
`kubectl delete -f client-pod.yml`

## More info on pods
`kubectl get pods -o wide`

## How to get info on deployments
`kubectl get deployments`