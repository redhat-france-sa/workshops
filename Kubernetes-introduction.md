> This is an excerpt from the excellent [Red Hat Developer Kubernetes Tutorial](https://redhat-scholars.github.io/kubernetes-tutorial).
> Check the original content if you got more than 1h30 in front of you :wink:

## Pre-requisite

* Get a Red Hat Developer account at https://developers.redhat.com
* Initialize your Developer Sandbox at https://developers.redhat.com/developer-sandbox/get-started

## Preparation

In your Developer Sandbox, open the **Web Terminal** tool on the right of header bar. Once connected in the terminal, initialized the current Kubernetes context on your `<username>-dev` namespace.

```sh
kubectl config set-context --current --namespace=laurent-broudoux-dev
```

## Pod, ReplicaSet, Deployment

### Pod

Start creating a raw pod:

```sh
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: quarkus-demo
spec:
  containers:
  - name: quarkus-demo
    image: quay.io/rhdevelopers/quarkus-demo:v1
EOF
```

Watch the pod lifecycle:

```sh
watch kubectl get pods
```

Verify the application in the Pod:

```sh
kubectl exec -it quarkus-demo /bin/sh
```

Run the next command. Notice that as you are inside the container instance, the hostname is `localhost`.

```sh
curl localhost:8080
```

Exit the current pod and delete it:

```sh
kubectl delete pod quarkus-demo
```

### Replicaset

Create a ReplicaSet:

```sh
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: ReplicaSet
metadata:
    name: rs-quarkus-demo
spec:
    replicas: 3
    selector:
       matchLabels:
          app: quarkus-demo
    template:
       metadata:
          labels:
             app: quarkus-demo
             env: dev
       spec:
          containers:
          - name: quarkus-demo
            image: quay.io/rhdevelopers/quarkus-demo:v1
EOF
```

Get the pods with labels:

```sh
watch kubectl get pods --show-labels
```

Inspect the ReplicaSet with following commands:

```sh
$ kubectl get rs
$ kubectl describe rs rs-quarkus-demo
```

Pods are "owned" by the ReplicaSet:

```sh
kubectl get pod rs-quarkus-demo-mlnng -o json | jq ".metadata.ownerReferences[]"
```

Now delete a pod, while watching pods. For that start another terminal first:

```sh
kubectl delete pod rs-quarkus-demo-mlnng
```

Check what's going on by listing the pods.

Delete the ReplicaSet to remove all the associated pods:

```sh
kubectl delete rs rs-quarkus-demo
```

### Deployment

Create a new Deployment

```sh
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: quarkus-demo-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: quarkus-demo
  template:
    metadata:
      labels:
        app: quarkus-demo
        env: dev
    spec:
      containers:
      - name: quarkus-demo
        image: quay.io/rhdevelopers/quarkus-demo:v1
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
EOF
```

Try following commands:

```sh
$ kubectl get pods --show-labels
$ kubectl exec -it quarkus-demo-deployment-5979886fb7-c888m -- curl localhost:8080
```

### Exposing an application

### Service

> This follows the creation of the Deployment in the previous chapter

Make sure you have Deployment, ReplicaSet and Pods:

```sh
$ kubectl get deployments
$ kubectl get rs
$ kubectl get pods
```

Create a Service:

```sh
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: the-service
spec:
  selector:
    app: quarkus-demo
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: LoadBalancer
EOF
```

Watch the Service creation until you see an external IP assigned.

```sh
watch kubectl get services
```

From the web terminal, try issuing following commands (replace <username> in the 2nd):

```sh
curl the-service:80
curl the-service.<username>-dev.svc.cluster.local:80
```

In the DevSandbox deployed on AWS, a loadbalancer ingress is automatically created for you. You need to retrieve the `hostname`:

```sh
kubectl get service the-service -o jsonpath="{.status.loadBalancer.ingress[0].hostname}"
```

Using this hostname, try out a single `curl` or a browser access from outside of the DevSandbox.

### Ingress

Now create an Ingress that reuses our service as a backend. Be sure to adapt the `host` below to your username and Sandbox cluster name:
  
```sh
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: the-service-ing
spec:
  rules:
  - host: the-service-ing-laurent-broudoux-dev.apps.sandbox.x8i5.p1.openshiftapps.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: the-service
            port:
              number: 80
EOF
```

And try accesing the host once created:

```sh
curl the-service-ing-laurent-broudoux-dev.apps.sandbox.x8i5.p1.openshiftapps.com
```
  
### OpenShift Route

Here, we're going to use the OpenShift client `oc` to build an OpenShift specific Route object:
  
```sh
oc expose service the-service  
```
  
Try the following commands:
  
```
kubectl get routes
oc get routes
```

You can use the `jq` utility to get the exact information you need from the command line:
  
```sh
oc get route the-service -o json | jq '.spec.host'
```
  
## Logs

There are various "production-ready" ways to do log gathering and viewing across a Kubernetes/OpenShift cluster. Many folks like some flavor of ELK (ElasticSearch, Logstash, Kibana) or EFK (ElasticSearch, FluentD, Kibana).

The focus here is on things a developer needs to get access to do in order to help understand the behavior of their application running inside of a pod.
  
> This follows the creation of the Deployment in the previous chapter
  
```sh
kubectl logs quarkus-demo-deployment-7cc99f8cf5-rrl7z
```
  
You can also follow the logs with the `-f` flag:
  
```sh
kubectl logs quarkus-demo-deployment-7cc99f8cf5-rrl7z -f
```
  
## Service Magic

### Deploy mypython

```sh
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mypython-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mypython
  template:
    metadata:
      labels:
        app: mypython
    spec:
      containers:
      - name: mypython
        image: quay.io/rhdevelopers/mypython:v1
        ports:
        - containerPort: 8000
EOF
```
  
### Deploy mygo

```sh
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mygo-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mygo
  template:
    metadata:
      labels:
        app: mygo
    spec:
      containers:
      - name: mygo
        image: quay.io/rhdevelopers/mygo:v1
        ports:
        - containerPort: 8000
EOF
```
  
### Deploy mynode

```sh
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mynode-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mynode
  template:
    metadata:
      labels:
        app: mynode
    spec:
      containers:
      - name: mynode
        image: quay.io/rhdevelopers/mynode:v1
        ports:
        - containerPort: 8000
EOF
```

Now wait for all pods to have the `Running` status:
  
```sh
watch kubectl get pods --show-labels
```
  
Create a new Service:
  
```sh
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: my-service
  labels:
    app: mystuff
spec:
  ports:
  - name: http
    port: 8000
  selector:
    inservice: mypods
  type: LoadBalancer
EOF
```

And explore these commands results:

```sh
kubectl describe service my-service
kubectl get endpointskubectl get endpoints my-service -o json | jq '.subsets[].addresses[].ip'
```

Last commands returns an error because we have no endpoints (Pods) attached to our service.

Because on the DevSandbox deployed on AWS, a loadbalancer ingress is automatically created for you. You need to retrieve the `hostname`:

```sh
SVC=$(kubectl get service my-service -o jsonpath="{.status.loadBalancer.ingress[0].hostname}")
```

Start a loop script that will poll the Service:

```sh
while true
do curl $SVC
sleep 0.8
done
```

```
# --- OUTPUT --- 
curl: (7) Failed to connect to 35.224.233.213 port 8000: Connection refused
curl: (7) Failed to connect to 35.224.233.213 port 8000: Connection refused
```

Now open a new terminal and start applying labels on pods, one by one:

```sh
kubectl label pod -l app=mypython inservice=mypods
kubectl label pod -l app=mynode inservice=mypods
kubectl label pod -l app=mygo inservice=mypods
```

You should see some changes in the output of loop script:
```sh
Python Hello on mypython-deployment-6874f84d85-2kpjl
Python Hello on mypython-deployment-6874f84d85-2kpjl
Node Hello on mynode-deployment-fb5457c5-hhz7h 0
Node Hello on mynode-deployment-fb5457c5-hhz7h 1
Python Hello on mypython-deployment-6874f84d85-2kpjl
Python Hello on mypython-deployment-6874f84d85-2kpjl
Python Hello on mypython-deployment-6874f84d85-2kpjl
```

Now checks the available endpoints for the Service as well as the Pods IP:

```sh
kubectl get endpoints my-service -o json | jq '.subsets[].addresses[].ip'
kubectl get pods -o wide
```
  
## Resource and Limits
  
Start creating a new deployment without any Requests or Limits:

```sh
kubectl apply -f https://github.com/redhat-scholars/kubernetes-tutorial/raw/master/apps/kubefiles/myboot-deployment.yml
```

Check that the default resources and limits are applied:

```sh
$ kubectl describe $(kubectl get pod -l app=myboot --field-selector 'status.phase!=Terminating' -o name)
# On OpenShift Dev sandbox
Containers:
  myboot:
    Limits:
      cpu:     1
      memory:  750Mi
    Requests:
      cpu:        10m
      memory:     64Mi
QoS Class: Burstable

# Otherwise...
Containers:
  myboot:
QoS Class: BestEffort
```

Check that default configuration on OpenShift Dev sandbox:

```sh
$ kubectl get limitrange resource-limits -o yaml
# On OpenShift Dev sandbox
apiVersion: v1
kind: LimitRange
metadata:
  annotations:
    toolchain.dev.openshift.com/last-applied-configuration: '{"apiVersion":"v1","kind":"LimitRange","metadata":{"labels":{"toolchain.dev.openshift.com/owner":"laurent-broudoux","toolchain.dev.openshift.com/provider":"codeready-toolchain"},"name":"resource-limits","namespace":"laurent-broudoux-dev"},"spec":{"limits":[{"default":{"cpu":"1000m","memory":"750Mi"},"defaultRequest":{"cpu":"10m","memory":"64Mi"},"type":"Container"}]}}'
  creationTimestamp: "2022-03-28T08:42:21Z"
  labels:
    toolchain.dev.openshift.com/owner: laurent-broudoux
    toolchain.dev.openshift.com/provider: codeready-toolchain
  name: resource-limits
  namespace: laurent-broudoux-dev
  resourceVersion: "999353061"
  uid: dafa4a62-8ddd-4892-af54-c35dbd49a246
spec:
  limits:
  - default:
      cpu: "1"
      memory: 750Mi
    defaultRequest:
      cpu: 10m
      memory: 64Mi
    type: Container
```

Create a service to easily interact with container:

```sh
$ kubectl apply -f https://github.com/redhat-scholars/kubernetes-tutorial/raw/master/apps/kubefiles/myboot-service.yml
```

Check it's running and explore resource perception:

```sh
$ curl myboot:8080      
Bonjour from Spring Boot! 1 on myboot-5dbffd5554-p2wzp

$ curl myboot:8080/sysresources
Memory: 7042 Cores: 8
```

> Bonus: explain this difference in memory perception ☕ 😉

Now see what happen when a pod is consuming all the resources:

```
# On another terminal
$ kubectl logs myboot-5dbffd5554-p2wzp -f

# On primary terminal
$ curl myboot:8080/consume     
curl: (52) Empty reply from server

# On the other terminal where 'kubectl logs myboot-5dbffd5554-p2wzp -f' runs
[...]
2022-03-28 10:07:34.069  INFO 7 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : FrameworkServlet 'dispatcherServlet': initialization completed in 11 ms
/consume myboot-5dbffd5554-p2wzp
Killed
```

## Rolling updates

> This follows the creation of the Deployment in the previous chapter

In a terminal, launch this loop script (if not already running):

```sh
while true
do curl myboot:8080
sleep 0.8
done
```

In another terminal, scale the number of replicas of `myboot` deployment:

```sh
kubectl scale deployments/myboot --replicas=2
```

Check you have 2 pods ready before going further. 

Now change the version of container image in the `myboot` deployment. You can do that through the OpenShift console, editing the YAML of with following command:

```sh
kubectl edit deployment myboot.
```

replacing `quay.io/rhdevelopers/myboot:v1` with `quay.io/rhdevelopers/myboot:v2` in the section below:

```yaml
    spec:
      containers:
      - image: quay.io/rhdevelopers/myboot:v2
        imagePullPolicy: IfNotPresent
        name: myboot
```

Check the ReplicaSets in your namespace:

```sh
$ kubectl get rs
NAME                                   DESIRED   CURRENT   READY   AGE
myboot-5dbffd5554                      0         0         0       26m
myboot-794c47599                       2         2         2       47s
```

and check out the events section of deployment description:

```sh
$ kubectl describe deployment myboot
[...]
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  28m    deployment-controller  Scaled up replica set myboot-5dbffd5554 to 1
  Normal  ScalingReplicaSet  5m33s  deployment-controller  Scaled up replica set myboot-5dbffd5554 to 2
  Normal  ScalingReplicaSet  4m6s   deployment-controller  Scaled up replica set myboot-794c47599 to 1
  Normal  ScalingReplicaSet  4m2s   deployment-controller  Scaled down replica set myboot-5dbffd5554 to 1
  Normal  ScalingReplicaSet  4m2s   deployment-controller  Scaled up replica set myboot-794c47599 to 2
  Normal  ScalingReplicaSet  3m15s  deployment-controller  Scaled down replica set myboot-5dbffd5554 to 0
```

You can list the revisions associated to your deployment by running the following command:

```sh
kubectl rollout history deployment/myboot
```

You can rollback to v1 using the following command:

```sh
kubectl rollout undo deployment/myboot --to-revision=1
```

In your terminal, you may have seen error messages like:

```
curl: (7) Failed to connect to 192.168.99.100 port 31528: Connection refused
```

The reason is the the missing Live and Ready Probes

Try using the Quarkus image instead of the Spring Boot one

```sh
kubectl set image deployment/myboot myboot=quay.io/rhdevelopers/quarkus-demo:v1
```

And there should be no errors, Quarkus simply boots up crazy fast. But be sure to add probes below!

## Liveness, Readiness & Startup

> This follows the creation of the Deployment in the previous chapter

Start updating this deployment with one containing liveness and readiness probes definitions:

```sh
kubectl apply -f https://github.com/redhat-scholars/kubernetes-tutorial/raw/master/apps/kubefiles/myboot-deployment-live-ready.yml
```

Check the probes definitions:

```sh
$ kubectl describe deployment myboot
[...]
Containers:
   myboot:
    Liveness:     http-get http://:8080/alive delay=10s timeout=2s period=5s #success=1 #failure=3
    Readiness:    http-get http://:8080/health delay=10s timeout=1s period=3s #success=1 #failure=3
```

Make sure a Service is deployed:

```sh
kubectl apply -f apps/kubefiles/myboot-service.yml
```

In a terminal, launch this loop script (if not already running):

```sh
while true
do curl myboot:8080
sleep 0.8
done
```

Check the output messageIn another terminal, scale the deployment to inspect Crash Loop Back errors because of badly tuned probes for OpenShift Dev Sandbox:

```sh
kubectl set image deployment/myboot myboot=quay.io/rhdevelopers/myboot:v2
```

You should notice the error free rolling update:

```
Aloha from Spring Boot! 134 on myboot-845968c6ff-9wvt9
Bonjour from Spring Boot! 0 on myboot-8449d5468d-m88z4
Bonjour from Spring Boot! 1 on myboot-8449d5468d-m88z4
```

> You may encounter Crash Loop Back errors because of badly tuned probes for OpenShift Dev Sandbox, try scaling down some other deployment to free resources in this case.

### Readiness

Once you've got a Pod running, you can check the utility of the probe:

```sh
$ kubectl exec -it myboot-5f45485689-pqprl /bin/bash
1024770000@myboot-5f45485689-pqprl:/app$ curl localhost:8080/misbehave
Misbehaving
1024770000@myboot-5f45485689-pqprl:/app$ exit
```

The pod is marked as NotReady and no longer receive requests. You can unstuck it with:

```sh
$ kubectl exec -it myboot-5f45485689-pqprl /bin/bash
1024770000@myboot-5f45485689-pqprl:/app$ curl localhost:8080/behave
Ain't Misbehaving
1024770000@myboot-5f45485689-pqprl:/app$ exit
```

### Liveness

Update the version of the deployment with:

```sh
kubectl set image deployment/myboot myboot=quay.io/rhdevelopers/myboot:v3
```

Once you see following messages in the loop terminal, your pod is ready:

```
Jambo from Spring Boot! 1 on myboot-5f8dbf44db-qrxhl
```

Now shot the pod making its liveness to return errors:

```sh
$ kubectl exec -it myboot-5f8dbf44db-qrxhl /bin/bash
1024770000@myboot-5f8dbf44db-qrxhl:/app$ curl localhost:8080/shot
I have been shot in the head
1024770000@myboot-5f8dbf44db-qrxhl:/app$
```

Wait some seconds or minutes and see the pod restarting. Your command terminal above should have been updated with:

```
1024770000@myboot-5f8dbf44db-qrxhl:/app$ command terminated with exit code 137
```

### Startup

Some applications require an additional startup time on their first initialization.

It might be tricky to fit this scenario into the liveness/readiness probes as you need to configure them for their normal behaviour to detect abnormalities during the running time and moreover covering the long start up time.

For instance, what if we had an application that might deadlock and we want to catch such issues immediately, we might have liveness and readiness probes that look like in https://github.com/redhat-scholars/kubernetes-tutorial/raw/master/apps/kubefiles/myboot-deployment-live-ready-aggressive.yml.

Apply a new deployment and check it allows you to scale:

```sh
kubectl apply -f https://github.com/redhat-scholars/kubernetes-tutorial/raw/master/apps/kubefiles/myboot-deployment-startup-live-ready.yml
kubectl scale deployments/myboot --replicas=2
```

## Environment & ConfigMap

Environment variables can be added to deployment to contextualize it.

```sh
$ kubectl set env deployment/myboot GREETING="Namaste"

$ kubectl describe deployment/myboot
```

ConfigMap is the Kubernetes resource that allows you to externalize your application’s configuration.

Create a new ConfigMap:

```sh
kubectl create cm my-config --from-literal=GREETING=Salut --from-literal=LOVE=Amour
```

Check its description:

```
kubectl describe cm my-config
```

Mount the ConfigMap as a volume:

```
oc set volume deployment/myboot --add --name=demo-volume --mount-path=/tmp/demo --type=configmap --configmap-name=my-config
```

Check consequences on `myboot` deployment.
