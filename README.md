# Kubernetes Response Engine based on Event-Driven Workflow using Argo Events & Argo Workflows

In earlier versions of a concept called _Kubernetes Response Engine_, we have used serverless platforms which run on top of Kubernetes such as Kubeless, OpenFaaS, and Knative. In a nutshell, this engine aims to provide a pipeline to users that they can take an action againts alerts which detected by Falco.

If you want to get more detail about the concept and how we have used serverless platforms with this concept, please follow the links below:

> * [Kubernetes Response Engine, Part 1 : Falcosidekick + Kubeless](https://falco.org/blog/falcosidekick-reponse-engine-part-1-kubeless/)
> * [Kubernetes Response Engine, Part 2 : Falcosidekick + OpenFaas](https://falco.org/blog/falcosidekick-reponse-engine-part-2-openfaas/)

But, one day [Edvin](https://github.com/NissesSenap) came with a great idea and asked can we use Cloud Native CI/CD systems to implement that same kind of scenario. Then, he used _Tekton_ and _Tekton Trigger_ to implement a _Kubernetes Response Engine_, to get more detail about his work, please follow this [repository](https://github.com/NissesSenap/falcosidekick-tekton).

After that, we realized that we can use _Argo Events_ and _Argo Workflows_ to do the same thing.This repository aims to provide an overview about how we can use these tools to implement _Kubernetes Response Engine_

Let's start with quick introduction of the tooling.

# What is Falco? [¶](github.com/falcosecurity/falco)
Falco, the open source cloud native runtime security project, is one of the leading open source Kubernetes threat detection engines. Falco was created by Sysdig in 2016 and is the first runtime security project to join CNCF as an incubation-level project.

# What is Falcosidekick? [¶](https://github.com/falcosecurity/falcosidekick) 
A simple daemon for enhancing available outputs for Falco. It takes a Falco's event and forwards it to different outputs.

# What is Argo Workflows? [¶](https://argoproj.github.io/argo-workflows/#what-is-argo-workflows)
Argo Workflows is an open source container-native workflow engine for orchestrating parallel jobs on Kubernetes. Argo Workflows is implemented as a Kubernetes CRD (Custom Resource Definition).

# What is Argo Events? [¶](https://argoproj.github.io/argo-events/#what-is-argo-events)
Argo Events is an event-driven workflow automation framework for Kubernetes which helps you trigger K8s objects, Argo Workflows, Serverless workloads, etc. on events from variety of sources like webhook, s3, schedules, messaging queues, gcp pubsub, sns, sqs, etc.

# Prerequisites

* minikube v1.19.0
* helm v3.5.4+g1b5edb6
* kubectl v1.21.0
* argo v3.0.2
* ko v0.8.2

# Demo

Let's start with explaining a little bit what we want to achieve in this demo. Basically, Falco the container runtime security tool is going to detect a unexpected behaviour on the host, then triggers an alert and send it to the Falcosidekick, Falcosidekick has _Webhook_ output type that we can configure, we configure this webhook output type to be able to notify webhook type of event source of the _Argo Events_, then _Argo Events_ trigger the [argoWorkFlowTrigger](https://github.com/argoproj/argo-events/blob/master/api/sensor.md#argoproj.io/v1alpha1.ArgoWorkflowTrigger) type of trigger of the _Argo Workflows_, and this workflow will create a _pod delete_ container, and this container will delete the Pod that cause an unexpected behaviour.

Falco --> Falcosidekick W/webhook --> Argo Events W/webhook --> Argo Workflows W/argoWorkFlowTrigger

Now, let's start with creating our local Kubernetes cluster.
```bash
$ minikube start
```

Then, install _Argo Events_ and _Argo Workflows_ components.
```bash
# Argo Events Installation
$ kubectl create namespace argo-events
namespace/argo-events created

$ kubectl apply \
    --filename https://raw.githubusercontent.com/argoproj/argo-events/stable/manifests/install.yaml
customresourcedefinition.apiextensions.k8s.io/eventbus.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/eventsources.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/sensors.argoproj.io created
serviceaccount/argo-events-sa created
clusterrole.rbac.authorization.k8s.io/argo-events-aggregate-to-admin created
clusterrole.rbac.authorization.k8s.io/argo-events-aggregate-to-edit created
clusterrole.rbac.authorization.k8s.io/argo-events-aggregate-to-view created
clusterrole.rbac.authorization.k8s.io/argo-events-role created
clusterrolebinding.rbac.authorization.k8s.io/argo-events-binding created
deployment.apps/eventbus-controller created
deployment.apps/eventsource-controller created
deployment.apps/sensor-controller created

$ kubectl --namespace argo-events apply \
    --filename https://raw.githubusercontent.com/argoproj/argo-events/stable/examples/eventbus/native.yaml
eventbus.argoproj.io/default created

# Argo Workflows Installation
$ kubectl create namespace argo
namespace/argo created

$ customresourcedefinition.apiextensions.k8s.io/clusterworkflowtemplates.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/cronworkflows.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/workfloweventbindings.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/workflows.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/workflowtemplates.argoproj.io created
serviceaccount/argo created
serviceaccount/argo-server created
serviceaccount/github.com created
role.rbac.authorization.k8s.io/argo-role created
role.rbac.authorization.k8s.io/argo-server-role created
role.rbac.authorization.k8s.io/submit-workflow-template created
role.rbac.authorization.k8s.io/workflow-role created
clusterrole.rbac.authorization.k8s.io/argo-clusterworkflowtemplate-role created
clusterrole.rbac.authorization.k8s.io/argo-server-clusterworkflowtemplate-role created
clusterrole.rbac.authorization.k8s.io/kubelet-executor created
rolebinding.rbac.authorization.k8s.io/argo-binding created
rolebinding.rbac.authorization.k8s.io/argo-server-binding created
rolebinding.rbac.authorization.k8s.io/github.com created
rolebinding.rbac.authorization.k8s.io/workflow-default-binding created
clusterrolebinding.rbac.authorization.k8s.io/argo-clusterworkflowtemplate-role-binding created
clusterrolebinding.rbac.authorization.k8s.io/argo-server-clusterworkflowtemplate-role-binding created
clusterrolebinding.rbac.authorization.k8s.io/kubelet-executor-default created
configmap/artifact-repositories created
configmap/workflow-controller-configmap created
secret/argo-postgres-config created
secret/argo-server-sso created
secret/argo-workflows-webhook-clients created
secret/my-minio-cred created
service/argo-server created
service/minio created
service/postgres created
service/workflow-controller-metrics created
deployment.apps/argo-server created
deployment.apps/minio created
deployment.apps/postgres created
deployment.apps/workflow-controller created
```

Let's verify if everything is working before we move on to the next step.
```bash
$ kubectl get pods --namespace argo-events
NAME                                      READY   STATUS    RESTARTS   AGE
eventbus-controller-7666b44ff7-k8bjf      1/1     Running   0          6m6s
eventbus-default-stan-0                   2/2     Running   0          5m33s
eventbus-default-stan-1                   2/2     Running   0          5m21s
eventbus-default-stan-2                   2/2     Running   0          5m19s
eventsource-controller-7fd7674cb4-jj9sn   1/1     Running   0          6m6s
sensor-controller-59b64579c9-5fbrv        1/1     Running   0          6m6s

$ kubectl get pods --namespace argo
NAME                                  READY   STATUS    RESTARTS   AGE
argo-server-5b86d9f84b-zl5nj          1/1     Running   3          5m32s
minio-58977b4b48-dnnwx                1/1     Running   0          5m32s
postgres-6b5c55f477-dp9n2             1/1     Running   0          5m32s
workflow-controller-d9cbfcc86-tm2kf   1/1     Running   2          5m32s
```
Let's install Falco and Falcosidekick.
```bash
$ helm upgrade --install falco falcosecurity/falco \
--namespace falco \
--create-namespace \
-f hacks/values.yaml

Release "falco" does not exist. Installing it now.
NAME: falco
LAST DEPLOYED: Thu May  6 22:43:52 2021
NAMESPACE: falco
STATUS: deployed
REVISION: 1
NOTES:
Falco agents are spinning up on each node in your cluster. After a few
seconds, they are going to start monitoring your containers looking for
security issues.


No further action should be required.
```

Let's verify if everything is working before move on to the next step.
```bash
$ kubectl get pods --namespace falco
NAME                                  READY   STATUS    RESTARTS   AGE
falco-falcosidekick-9f5dc66f5-nmfdp   1/1     Running   0          68s
falco-falcosidekick-9f5dc66f5-wnm2r   1/1     Running   0          68s
falco-zwxwz                           1/1     Running   0          68s
```

Now, we are good to go to set up our workflow by creating event source and the trigger.
```bash
# Create event source
$ kubectl apply -f webhooks/webhook.yaml
eventsource.argoproj.io/webhook created

$ kubectl get eventsources --namespace argo-events
NAME      AGE
webhook   11s

$ kubectl get pods --namespace argo-events
NAME                                         READY   STATUS    RESTARTS   AGE
eventbus-controller-7666b44ff7-k8bjf         1/1     Running   0          18m
eventbus-default-stan-0                      2/2     Running   0          17m
eventbus-default-stan-1                      2/2     Running   0          17m
eventbus-default-stan-2                      2/2     Running   0          17m
eventsource-controller-7fd7674cb4-jj9sn      1/1     Running   0          18m
sensor-controller-59b64579c9-5fbrv           1/1     Running   0          18m
webhook-eventsource-z9bg6-6769c7bbc8-c6tff   1/1     Running   0          45s # <-- Pod listens webhook event.

# necessary RBAC permissions for trigger and the pod-delete container
$ kubectl apply -f hacks/workflow-rbac.yaml
serviceaccount/operate-workflow-sa created
clusterrole.rbac.authorization.k8s.io/operate-workflow-role created
clusterrolebinding.rbac.authorization.k8s.io/operate-workflow-role-binding created

$ kubectl apply -f hacks/delete-pod-rbac.yaml
serviceaccount/falco-pod-delete created
clusterrole.rbac.authorization.k8s.io/falco-pod-delete-cluster-role created
clusterrolebinding.rbac.authorization.k8s.io/falco-pod-delete-cluster-role-binding created

# Create trigger
$ kubectl apply -f sensors/sensors-workflow.yaml
sensor.argoproj.io/webhook created

$ kubectl get sensors --namespace argo-events
NAME      AGE
webhook   5s
 
$ kubectl get pods --namespace argo-events
NAME                                         READY   STATUS    RESTARTS   AGE
eventbus-controller-7666b44ff7-k8bjf         1/1     Running   0          25m
eventbus-default-stan-0                      2/2     Running   0          25m
eventbus-default-stan-1                      2/2     Running   0          25m
eventbus-default-stan-2                      2/2     Running   0          25m
eventsource-controller-7fd7674cb4-jj9sn      1/1     Running   0          25m
sensor-controller-59b64579c9-5fbrv           1/1     Running   0          25m
webhook-eventsource-z9bg6-6769c7bbc8-c6tff   1/1     Running   0          8m11s
webhook-sensor-44w7w-7dcb9f886d-bnh8f        1/1     Running   0          74s # <- Pod will create workflow.
```

> I'm using google/ko project to build and push container image, but you don't have to do this, I already built an image and pushed it to the registry, but if you want to build your own image install google/ko project and run the command below and change the image version inside sensors/sensors-workflow.yaml                              $ KO_DOCKER_REPO=devopps ko publish . --push=true -B

There is one more thing left that we need to do, this is installing [argo CLI](https://argoproj.github.io/argo-workflows/cli/) for worklow.
```bash
$ # Download the binary
curl -sLO https://github.com/argoproj/argo/releases/download/v3.0.2/argo-darwin-amd64.gz

# Unzip
gunzip argo-darwin-amd64.gz

# Make binary executable
chmod +x argo-darwin-amd64

# Move binary to path
mv ./argo-darwin-amd64 /usr/local/bin/argo

# Test installation
argo version
```

Now, let's test the whole environment. We are going to create an alpine based container, then we'll exec into it,once we exec into the container, Falco will detect it. After that we should see the status of the Pod as _Terminating_.
```bash
$ kubectl run alpine --namespace default --image=alpine --restart='Never' -- sh -c "sleep 600"
pod/alpine create

$ kubectl exec -i --tty alpine --namespace default -- sh -c "uptime"
```

You should see the similar outputs like the following screen:

![screen_shot](./screenshot.png)

