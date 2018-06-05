# Kubernetes workshop 5th Jun 2018
This repo contains the files, documentation and exercises used in the Kubernetes workshop held at adidas HQ in Amsterdam on the 5th of June 2018.

---

# Introduction
The goals of this workshop are:

* Understand what is Kubernetes
* Understand the most commonly used concepts/terms associated with k8s
* Learn how to write k8s manifests
* Learn how to handle deployments and upgrades of an application
* Learn how an application is scaled manually or dynamically

# Getting started
Make sure you have `docker`, `minikube` and `kubectl` installed.

* Fork this repository, `https://github.com/lauriku/k8s-workshop.git`
* Clone it

## Running the container locally
* `docker build -t lauriku/k8s-workshop .`
* `docker run -it --rm -p 3000:3000 lauriku/k8s-workshop`
* Browse [localhost:3000](http://localhost:3000)

## (For extra credit!) Creating a docker hub account

* Create a docker hub account
* Create an automated build
* Image can now be pulled by `docker pull <username>/k8s-workshop:latest`

---

# Prep local environment
## minikube
minikube is an implementation of a local Kubernetes cluster, that can be used when testing manifests, deployments and so on.
* BE WARNED: minikube actually overwrites your existing kubeconfig, so take a backup of your `~/.kube/config` if you wish to preserve it.
* Run `minikube start` to get the cluster running
* `kubectl` now has configuration pointing to this local cluster, for this terminal session
* Try running `kubectl cluster-info`
* Double check what cluster you are pointing at if you use an another terminal session
* Enable the ingress controller, so we can test creating ingresses, by running `minikube addons enable ingress`
* Enable metrics collection by running `minikube addons enable heapster`

---
# Workshop Exercises
The first thing to do is to write manifests for the Kubernetes manifest is a description of a _desired state_ of a resource. Three different resource types are going to be needed for this workshop, `deployment`, `service` and `ingress`.

Templates for the manifests can be located under the `deploy/` folder.

## 1. deployment.yml

a. Give the `deployment` resource a name. This can then be used to later access the resource, to update or delete it for example.

```yaml
metadata:
  name: lauriku-app
```

b. Start writing the `template: spec` for the `deployment`. This defines the Pod template that describes which containers are to be launched. In order for other resources to route traffic to the pods, the pods need a `label`.
```yaml
spec:
  template:
    metadata:
      labels:
        app: lauriku-app
```

c. Next, define the containers to be run in these pods. The container needs a `name`, `image`and a `containerPort`. The `image` field refers to the image now stored in the docker hub repository. The `containerPort` should the same port that container exposes, and the process inside the container listens to.

```yaml
spec:
  template:
  ...
    spec:
      containers:
        - name: lauriku-app
          image: lauriku/k8s-workshop:latest
          ports:
            - containerPort: 3000
```

Now, you should be able to send this manifest to the Kubernetes API, so that it can start building your application.

```bash
kubectl apply -f deploy/deployment.yml
```

You should be able to see the pods starting by writing `kubectl get pods`.

To see that the pod has started properly, you can check the logs with `kubectl logs <pod-name>`

## 2. service.yml

a. Like the `deployment`, the `service` needs a name and a label. The name is important here, as it will be used to link it with the ingress in the next step.

```yaml
metadata:
  name: lauriku-svc
  labels:
    app: lauriku-app
```

b. The `spec` of the `service` needs definitions on what port to map to which container, and what protocol to use. `targetPort` is the port that is exposed by the container, and where the service will direct traffic to. `port` can be any port, but for simplicity we'll use the same here.

```yaml
spec:
  ports:
    - port: 3000
      targetPort: 3000
      protocol: TCP
```

c. Lastly, the `service` needs a `selector`, to know which pods to direct traffic to.

```yaml
spec:
  ...
    selector:
      app: lauriku-app
```

The service manifest can be applied the same way as the deployment, so

```bash
kubectl apply -f deploy/service.yml
```

And `kubectl get service -o yaml` should now show detailed information of it.

## 3. ingress.yml

a. You know the drill, the `ingress` controller needs some identifying information, so let's go:

```yaml
metadata:
  name: lauriku-ing
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
```

b. Then the `spec`. For the `ingress`, this is a set of rules that determine what services traffic is routed to, based on routes for example.

```yaml
spec:
  rules:
  - http:
      paths:
      - path: /.*
        backend:
          serviceName: lauriku-svc
          servicePort: 3000
```

Here we just route all traffic to port 3000 of the `lauriku-svc` service.

Finally, apply this manifest with `kubectl apply -f deploy/ingress.yml`. It can then be inspected by `kubectl get ingress lauriku-ing -o yaml`.

### Setting up connectivity
In order to get the correct ip address and port the deployment from the `docker-machine` that is running `minikube`, run the following snippet, and it should open up a browser pointing to the ingress created in the previous step.

```bash
ENDPOINT_HOST=$(minikube ip)
open $ENDPOINT_HOST
```
---

## 4. Scaling the deployment
There are a few ways to do the scaling, the quickest being the `kubectl scale` command.

Try the following:

```bash
kubectl scale --replicas=4 deployment/lauriku-app && \
kubectl rollout status deployment/lauriku-app
```

This controls the `deployment` object in Kubernetes directly, but of course if there's a new `kubectl apply` from the repo, the changes would be overwritten. The `deployment.yml` can be updated with:

```yaml
spec:
  replicas: 4
```

And then applied with `kubectl apply -f deploy/deployment.yml`.

### Kubernetes dashboard
You can browse your resources from the Kubernetes dashboard as well, just run `minikube dashboard`.

## 5. Upgrading the deployment
Once we have an existing image, an upgrade can be performed by just editing the existing deployment. This can be done with the `kubectl set image` command.

But first, adjust the update policy for the deployment a bit. Add the following to the deployment manifest:

```yaml
spec:
  strategy:
      type: RollingUpdate
      rollingUpdate:
        maxSurge: 25%
        maxUnavailable: 25%
```

And then apply the policy with `kubectl apply -f deploy/deployment.yml`.

EXTRA CREDIT: Configure your Docker Hub automated builds to tag images based on git tags. Then, do a change to `index.js`, tag it, and wait for an image to build, and use the tag with the upgrade in the next step.

Then, you can fire up the RollingUpdate by setting the image of the deployment to the new one:

```bash
kubectl set image deployment/lauriku-app lauriku-app=lauriku/k8s-workshop:v2 && \
kubectl rollout status deployment/lauriku-app
```

## 6. Rolling back the deployment
Oopsie, you made a mistake. How do you roll back the deployment?

You can check the history of a deployment by `kubectl rollout history deployment/lauriku-app`, and inspect each revision with the `--revision` flag, so for example:

```bash
$ kubectl rollout history deployment/lauriku-app --revision=1
deployments "lauriku-app" with revision #1
Pod Template:
  Labels:	app=lauriku-app
	pod-template-hash=3495417412
  Containers:
   lauriku-app:
    Image:	lauriku/k8s-workshop:latest
    Port:	3000/TCP
    Host Port:	0/TCP
    Environment:	<none>
    Mounts:	<none>
  Volumes:	<none>
```

Looking here, we can see that _revision #1_ has the previous version of the image. So the rollback to this version could be done with: `kubectl rollout undo deployment/lauriku-app --to-revision=2`. Just saying `rollout undo deployment/<deployment_name>` without `--to-revision`, will perform a rollback to the previous version.

### Editing resources live
Instead of always editing the resources and then applying the changes, you can also edit them live if needed.

Try your hand in editing a manifest "live", by doing the following:

```bash
KUBE_EDITOR="vim" # or if you want to use VScode, 'code -w'
kubectl edit deployment/lauriku-app
```

## 7. Resource requests and limits
You can limit the resource usage of different pods, with resource `limits` and `requests`. Requests are used to reserve a certain amount of cpu/mem resources when a pod starts.

Add the following to the deployment manifest:

```yaml
spec:
  template:
    spec:
      containers:
        - name: lauriku-app
          ...
          resources:
          requests:
            memory: "128Mi"
            cpu: "5m"
          limits:
            memory: "256Mi"
            cpu: "10m"

```

And apply the manifest with `kubectl apply -f deploy/deployment.yml`.

## 8. Horizontal Pod Autoscaling

You can first check the Grafana dashboard to see a few different metrics in action, `minikube addons open heapster`.

For the HPA to work, we need a HorizontalPodAutoscaler resource. You can create this to `deploy/hpa.yml`.

```yaml
---
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: lauriku-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1beta1
    kind: Deployment
    name: lauriku-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 30
  - type: Resource
    resource:
      name: memory
      targetAverageUtilization: 50
```