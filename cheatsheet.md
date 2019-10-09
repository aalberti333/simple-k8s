## Basic up and running. Let's set sail! :boat:

## What is minikube?
A virtual environment you'll use to communicate with k8s when performing local development. You'll need virtualbox installed before installing minikube. Refer to the [docs](https://kubernetes.io/docs/tasks/tools/install-minikube/) for install instructions.

## What are pods?
Collection of containers that need each other to run. For example, Zookeeper and Kafka. Kafka is unable to start without Zookeeper, so they would need to be placed together in a pod.

## What are services?
Routing/port exposure/networking for the pods. These will differ depending on whether you're running tests in a development environment (using nodeports) vs. a production environment (using clusterIP with ingress).

## Okay cool, let's get started
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

## What to do after updating an image?
`kubectl apply` won't notice the change, so extra work needs to be done.

**Solutions:**
* Manually delete pods: seems silly, but works, a pod will be recreated and pull the latest version of the image. You also might make the mistake of deleting the wrong set of pods, which is a big whoops.

* Tagging of Docker images: works, but adds an extra step of updating the version number. You could use an environment variable for the tag, but *you can't use env variables in k8s config files*.

* Use an imperative command to update the image version the deployment should use: in command line, tell k8s to update the config file with the latest version of the image, eh... but it's the best of a bad situation. All of these solutions are not very...ideal, but this is considered the best. So, to be clear, we'll tag our docker image, then use the command line to tell k8s which version we want, which will change the config file for us.

Command to use new image version will be:
`kubectl set image deployment/client-deployment client=stephengrider/multi-client:v5`

or

`kubectl set image <type>/<name-of-type> <name-of-pod>=<image>:<tag>`

## Docker stuff
How can we reach into the node (minikube) and play around with the docker inside there?

Well, we can reconfigure the docker cli to talk to minikube
`eval $(minikube docker-env)`

then, run: `docker ps`

This will only reconfigure the docker cli in your *current terminal window*. Other windows won't be affected. This change isn't permanent, and is only specific to one terminal window.

This just sets up some environment variables to point docker to minikube.


## Nodeport vs. ClusterIP
Services have several networking options:
* ClusterIP: exposes a set of pods to _other objects in the cluster_ (like a docker-compose with a name of the service, but the port is not exposed). Does not allow traffic from the outside world. Traffic comes into the cluster through the **Ingress Service**
* NodePort: exposes a set of pods to the outside world (only good for dev purposes!! like a docker-compose with ports exposed)
* LoadBalancer: 
* Ingress: 

## Deploy multiple config files at once
Essentially the same as deploying a single file, but just list the directory instead:
`kubectl apply -f k8s`

## Why not have all my configs in one file?
Well, you can. Just add `---` to separate each configuration; however, separation of files lets you know how many objects you have and it's extremely clear to know what's-what if you named the files appropriately. Ultimately, it's a matter of preference.

## About the types of volumes
A quick note on replicas for databases: Having multiple replicas access the same volume without cooperation is a recipe for disaster. So, Postgres (and other databases) requires additional configuration when trying to scale up.

So what does _volume_ mean in the world of k8s? It means an object that allows a container to store data at the pod level. (This is a bit different than the generic terminology, which would be some type of mechanism that allows a container to access a filesystem outside itself).

Let's look at the 3 types:
* persistent volume claim (we want this)
* persistent volume (we want this)
* volume (we don't want this for data that needs to last, note exactly the same thing as a Docker volume!!)

### Why not use volume?
A k8s volume belongs to a specific pod. It can be accessed by anything inside that pod. The benefit? If a specific pod container dies and restarts inside the pod, then the new container has access to anything in the volume. However, the volume is tied to the pod, so if the pod ever dies/gets recreated/ terminated/ whatever, **the volume does too**. So, a volume is not appropriate for saving data in a database.

### Persistent volume
A persistent volume is long term durable storage that's not tied to a specific pod or a specific container. This way, if a pod crashes/recreated/whatever, the volume still lasts and can be reconnected to.

### Persistent volume claim
Let's say you have a pod configuration. You have multiple storage options for it. A PVC is an advertisement (can't store anything, just an advertisement) saying "here are the different options that you have access to for storage inside of this cluster". We will write these claims in the config files: "there should be these options for my config, this is something I can get for my pod when it's created". So when you ask for it, k8s looks through persistent volumes that are readily available (these are called statically provisioned persistent volumes). These were created ahead of time, that can be used right away for storage. If k8s can't find what you're looking for, the persistent volume is created on the fly (these are called dynamically provisioned persistent volumes, it's not created ahead of time, just when you ask for it).