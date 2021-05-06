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

# Demo

Let's start with explaining a little bit what we want to achieve in this demo. Basically, Falco the container runtime security tool is going to detect a unexpected behaviour on the host, then triggers an alert and send it to the Falcosidekick, Falcosidekick has _Webhook_ output type that we can configure, we configure this webhook output type to be able to notify webhook type of event source of the _Argo Events_, then _Argo Events_ trigger the [argoWorkFlowTrigger](https://github.com/argoproj/argo-events/blob/master/api/sensor.md#argoproj.io/v1alpha1.ArgoWorkflowTrigger) type of trigger of the _Argo Workflows_, and this workflow will create a _pod delete_ container, and this container will delete the Pod that cause an unexpected behaviour.

Falco --> Falcosidekick W/webhook --> Argo Events W/webhook --> Argo Workflows W/argoWorkFlowTrigger
