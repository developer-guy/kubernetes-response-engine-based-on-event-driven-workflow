# Kubernetes Response Engine based on Event-Driven Workflow using Argo Events & Argo Workflows

We presented in previous blog posts the concept called _Kubernetes Response Engine_, to do so we have used serverless platforms running on top of Kubernetes such as Kubeless, OpenFaaS, and Knative. In a nutshell, this engine aims to provide to users automatic action against threats detected by Falco.

If you want to get more details about the concept and how we use serverless platforms for this concept, please follow the links below:

> * [Kubernetes Response Engine, Part 1 : Falcosidekick + Kubeless](https://falco.org/blog/falcosidekick-reponse-engine-part-1-kubeless/)
> * [Kubernetes Response Engine, Part 2 : Falcosidekick + OpenFaas](https://falco.org/blog/falcosidekick-reponse-engine-part-2-openfaas/)
> * [Kubernetes Response Engine, Part 3 : Falcosidekick + Knative](https://falco.org/blog/falcosidekick-reponse-engine-part-3-knative/)

Recently, a community member, [Edvin](https://github.com/NissesSenap), came with the great idea to use a Cloud Native Workflow system to implement same kind of scenario. Following that, he implemented it by using _Tekton_ and _Tekton Trigger_. To get more details about his work, please follow this [repository](https://github.com/NissesSenap/falcosidekick-tekton).

After that, we realized that we can use _Argo Events_ and _Argo Workflows_ to do the same thing. This repository provides an overview of how we can use these tools to implement a _Kubernetes Response Engine_

Let's start with quick a introduction of the tooling.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [What is Falco? ¶](#what-is-falco-%C2%B6)
- [What is Falcosidekick? ¶](#what-is-falcosidekick-%C2%B6)
- [What is Argo Workflows? ¶](#what-is-argo-workflows-%C2%B6)
- [What is Argo Events? ¶](#what-is-argo-events-%C2%B6)
- [Prerequisites](#prerequisites)
- [Demo](#demo)
  - [Minikube](#minikube)
  - [Kind](#kind)
  - [Install Argo Events and Argo Workflows](#install-argo-events-and-argo-workflows)
  - [Install Falco and Falcosidekick](#install-falco-and-falcosidekick)
  - [Install Webhook and Sensor](#install-webhook-and-sensor)
  - [Install argo CLI](#install-argo-cli)
  - [Test](#test)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## What is Falco? [¶](github.com/falcosecurity/falco)

Falco, the open source cloud native runtime security project, is one of the leading open source Kubernetes threat detection engines. Falco was created by Sysdig in 2016 and is the first runtime security project to join CNCF as an incubation-level project.

## What is Falcosidekick? [¶](https://github.com/falcosecurity/falcosidekick)

A simple daemon for connection Falco to your ecosystem (alerting, logging, metrology, etc).

## What is Argo Workflows? [¶](https://argoproj.github.io/argo-workflows/#what-is-argo-workflows)

Argo Workflows is an open source container-native workflow engine for orchestrating parallel jobs on Kubernetes. Argo Workflows are declared through a Kubernetes CRD (Custom Resource Definition).

## What is Argo Events? [¶](https://argoproj.github.io/argo-events/#what-is-argo-events)

Argo Events is an event-driven workflow automation framework for Kubernetes which helps you trigger K8s objects, Argo Workflows, Serverless workloads, and others by events from variety of sources like webhook, s3, schedules, messaging queues, gcp pubsub, sns, sqs, etc.

## Prerequisites

* minikube v1.19.0 or kind v0.10.0
* helm v3.5.4+g1b5edb6
* kubectl v1.21.0
* argo v3.0.2
* ko v0.8.2

## Demo

Let's start with explaining a little bit what we want to achieve in this demo. Basically, Falco, the container runtime security tool, is going to detect an unexpected behaviour at host level, then it will trigger an alert and send it to Falcosidekick. Falcosidekick has _Webhook_ output type we can configure to notify the event source of _Argo Events_. Then, _Argo Events_  will trigger the [argoWorkFlowTrigger](https://github.com/argoproj/argo-events/blob/master/api/sensor.md#argoproj.io/v1alpha1.ArgoWorkflowTrigger) type of trigger of _Argo Workflows_, and this workflow will create a _pod delete_ pod in charge of terminating the compromised pod.

Falco --> Falcosidekick W/webhook --> Argo Events W/webhook --> Argo Workflows W/argoWorkFlowTrigger

Now, let's start with creating our local Kubernetes cluster.

### Minikube

```bash
minikube start
```

### Kind

If you rather use kind.

```shell
# kind config file
cat <<'EOF' >> kind-config.yaml.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  image: kindest/node:v1.20.2
  extraMounts:
    # allow Falco to use devices provided by the kernel module
  - hostPath: /dev
    containerPath: /dev
    # allow Falco to use the Docker unix socket
  - hostPath: /var/run/docker.sock
    containerPath: /var/run/docker.sock
- role: worker
  image: kindest/node:v1.20.2
  extraMounts:
    # allow Falco to use devices provided by the kernel module
  - hostPath: /dev
    containerPath: /dev
    # allow Falco to use the Docker unix socket
  - hostPath: /var/run/docker.sock
    containerPath: /var/run/docker.sock
- role: worker
  image: kindest/node:v1.20.2
  extraMounts:
    # allow Falco to use devices provided by the kernel module
  - hostPath: /dev
    containerPath: /dev
    # allow Falco to use the Docker unix socket
  - hostPath: /var/run/docker.sock
    containerPath: /var/run/docker.sock
EOF

# start the kind cluster

kind create cluster --config=./config-kind.yaml

```

> Kind is verified on on linux client only.

### Install Argo Events and Argo Workflows

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

$ kubectl apply -n argo -f https://raw.githubusercontent.com/argoproj/argo-workflows/stable/manifests/quick-start-postgres.yaml
customresourcedefinition.apiextensions.k8s.io/clusterworkflowtemplates.argoproj.io created
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

### Install Falco and Falcosidekick

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

Let's verify if all components for falco are up and running.

```bash
$ kubectl get pods --namespace falco
NAME                                  READY   STATUS    RESTARTS   AGE
falco-falcosidekick-9f5dc66f5-nmfdp   1/1     Running   0          68s
falco-falcosidekick-9f5dc66f5-wnm2r   1/1     Running   0          68s
falco-zwxwz                           1/1     Running   0          68s
```

### Install Webhook and Sensor

Now, we are ready to set up our workflow by creating the event source and the trigger.

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

> We use google/ko project to build and push container images, but you don't have to do this, we already built an image and pushed it to the registry. If you want to build your own image, install google/ko project and run the command below after having changed the image version inside sensors/sensors-workflow.yaml
> `KO_DOCKER_REPO=devopps ko publish . --push=true -B`

### Install argo CLI

There is one more thing we need to do, this is installation of [argo CLI](https://argoproj.github.io/argo-workflows/cli/) for managing worklows.

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

### Test

Now, let's test the whole environment. We are going to create an alpine based container, then we'll `exec`` into it. At moment we'll exec into the container, Falco will detect it and you should see the status of the Pod as _Terminating_.

```bash
$ kubectl run alpine --namespace default --image=alpine --restart='Never' -- sh -c "sleep 600"
pod/alpine created

$ kubectl exec -i --tty alpine --namespace default -- sh -c "uptime" # this will trigger the event
```

You should see the similar outputs like the following screen:

![screen_shot](./screenshot.png)
