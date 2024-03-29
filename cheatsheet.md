## Basic up and running. Let's set sail! :boat:

## What is minikube?
A virtual environment you'll use to communicate with k8s when performing local development. You'll need virtualbox installed before installing minikube. Refer to the [docs](https://kubernetes.io/docs/tasks/tools/install-minikube/) for install instructions.

## What is kubectl?
Command line tool which interacts with a k8s cluster. In our case, we'll be interacting with minikube. More info can be found in the [docs](https://kubernetes.io/docs/reference/kubectl/overview/)

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

## Talking to Docker inside minikube :whale:
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
* LoadBalancer: Legacy way of getting network traffic into a cluster
* Ingress: Exposes a set of services to the outside world 

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

#### Breaking down PVC config
* Access Modes have three types:
    * ReadWriteOnce: Can be used by a single node
    * ReadOnlyMany: Multiple nodes can read from this
    * ReadWriteMany can read and written to by many nodes

* Resources
    * Storage: amount of space needed

To check options for storage, you can run:
`kubectl get storageclass`

For more storage info, run
`kubectl describe storageclass`

Storage class options are [here](https://kubernetes.io/docs/concepts/storage/storage-classes/#provisioner). You'll usually use the default options, this is where k8s will look for storage. Our default is minikube, but in production you will likely be using one of these defaults and need to specify which one.

After applying, you can check these stats with:

`kubectl get pv`

`kubectl get pvc`

### Environment variables
Pretty similar to docker compose. You'll use a key named `env` in your container values which contains `name` and `value`. These need to be string values, so wrap any port numbers in `''`.

#### Secrets
Used to securely store a piece of info in the cluster, such as a database password. Unlike other objects (Pods, Deployments, Services), this will not be created with a config file. This instead will be created with an imperative command (Gotta keep it secret! Snitches get stiches).

Here's how to create one:

`kubectl create secret generic <secret_name> --from-literal key=value`

* `generic`: Type of secret (used vast majority of the time). Other types are `docker-registry` and `tls` (for https stuff)

* `--from-literal`: We're going to add the secret into this command, as opposed to from a file 

We're going to run:
`kubectl create secret generic pgpassword --from-literal PGPASSWORD=12345asdf`

Then use `kubectl get secrets` to verify.

#### Load Balancer
Legacy way of getting network traffic into a cluster.

Does a couple things. You would make a configuration file of type service and subtype load balancer. Setting this up allows access to *only one* specific deployment (one set of pods). Load balancer service will also reach out to your cloud provider, and create a load balancer using their configuration of what that is (whatever AWS, or Google Cloud defines that to be) for your chosen specifc deployment.

**We're not going to use this, supposedly it's deprecated**. Instead, we'll be using *Ingres*. This will allow access to multiple deployments.

#### Ingres
Several specific implementations of Ingress.

**We are using ingress-nginx, a community project led by k8s** [here](github.com/kubernetes/ingress-nginx)

**We will not be using kubernetes-ingres, a project led by the company nginx** [here](github.com/nginxinc/kubernetes-ingress)

BE CAREFUL ABOUT THIS! They are nearly identical. Again, we'll be using *ingress-nginx*

The setup of this ingress will be different depending on your cloud provider. We'll be setting it up:

1. On our local machine
2. On Google Cloud (more on this later)

### How it works (kinda...)
A **controller** is any type of object that constantly works to make some desired state a reality inside of our cluster. Ingress will work the same way, it has a controller. When we feed in this config file, the controller will create a pod running nginx that's going to have a particular set of rules to make sure traffic comes in and gets sent off to the appropriate different services inside of our cluster.

Ingress config -> Ingress controller (once fed into kubectl) -> creates something that accepts incoming traffic (for ingress-nginx, this will actually be the same thing as the controller. This may not be the case for other Ingress choices) -> sends traffic to multiple deployments.

#### Minikube Dashboard
Run `minikube dashboard` to see the dashboard! Pretty cool! Stuff can be created/deleted/edited from here, but it's not recommended (think about doing this with code on Github, it's better practice to do everything locally and send those changes up)

### Deploying
You can run a `deploy.sh` script instide your CI. You can find that [here](./simple_k8s/deploy.sh). Remember to *tag your images!* Without changing the tag, the image will not update. K8s will say "Hmm, you pushed a latest, but I'm already running latest, so I'm not going to do anything since these are the same thing". Creating a unique tag is required for k8s to update.

So what should we do? Tag it twice:

1. `<my-image>:latest`
2. `<my-image>:$GIT_SHA`

Where you will copy/paste your GIT_SHA in `$GIT_SHA`.

Using a unique SHA for every deploy will also allow you to roll back your deployed images if something ever goes wrong to test which deployment caused the issue.

#### So why still use "latest"?
Let's say a new engineer runs `kubectl apply -f k8s` to test development locally. This way, if the engineer wants to pull the newest version of the image, they won't have to know the SHA to get the newest version.

#### Getting the SHA
You can set an environment variable in CI by running: `SHA=$(git rev-parse HEAD)`.

#### Helm (and its friends)
[Helm](https://helm.sh/) helps you manage k8s applications. Helm Charts help you define, install, and upgrade even the most complex k8s application. (Charts are packages of pre-configured k8s resources). You would use Helm to install something like Ingress-NGINX.

It's essentially a tool that streamlines installing and managing k8s applications. Think of it like apt/yum/homebrew for k8s. It works directly with **Tiller**, a pod with Tiller will run inside our cluster, and will make changes to the inside of our cluster. It will try to install new sets of configs, new secrets, new sets of deployments, whatever else it might be. But to do this, Tiller needs permission to.

**RBAC** is Role Based Access Control. It limits who can access and modify objects in our cluster. Tiller wants to make changes to our cluster, so it needs to get some permission set. Some RBAC terminology:

* User Accounts: Identifies a *person* administering our cluster
* Service Accounts: Identifies a *pod* administering a cluster
* ClusterRoleBinding: Authorizes an account to do a certain set of actions across the entire cluster
* RoleBinding: Authorizes an account to do a certain set of actions in a *single namespace*

`kubectl create serviceaccount --namespace kube-system tiller`: Create a new service account called tiller in the kube-system namespace

`kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller`: Create a new clusterrolebinding with the role 'cluster-admin' and assign it to the service account 'tiller'

After running the above commands, we should be able to use Helm.

### Local development with Skaffold
Skaffold is a commandline tool designed to be used with k8s to be used for local development. Skaffold watches a project directory for changes, and when it sees a change, it jumps into action. It will take that change and get it reflected in the k8s cluster. It can do this in one of two modes:

1. Rebuild client image from scratch, update k8s
2. Inject updated files into a pod, rely on app to automatically update itself

In our case, because react has hot reload and nodemon is being used, method 2 will be used here.

Run `skaffold dev` in the same directory as the [skaffold.yml](./simple_k8s/skaffold.yml) file to run.

Skaffold will also clean-up its own environment for you, so long as you add them to the **manifests** section of [skaffold.yml](./simple_k8s/skaffold.yml). However, if you have anything persistent (like volumes/databases/etc) it's recommended not to add those in **manifests**.
