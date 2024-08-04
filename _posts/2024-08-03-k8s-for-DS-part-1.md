---
layout: post
title: A Kubernetes Primer for Data Science and ML Backgrounds - part 1 - Background and Setup
author: rob
---

At some point in your career, you’ll encounter Kubernetes. This is a primer on Kubernetes for people with data science or ML research backgrounds. I started my career as a software engineer, pivoted to ML engineering, and am now hooked on MLOps. One thing I’ve always admired about DS and ML colleagues is the willingness to pick up engineering tasks to get the job done. Based on that, I thought there might be an appetite for a Kubernetes primer for people with an ML background, one that’s practical but runs through enough of the theoretical groundwork to build the right intuition. You might have found yourself with failing pipelines that you want to debug without raising a support request. If so, this series of articles will teach you the crucial subset of Kubernetes commands you need to get to the bottom of an issue. You might even be co-founding a startup shouldering the MLOps work yourself (in which case, get started with Kubernetes in this post and then subscribe for a follow up series on bootstrapping a barebones MLOps setup for a small-scale experimentation driven workflow). 

# Prerequisites 

- You’re using a Mac or Linux based machine
- You have Homebrew installed 
- You know about containers and Docker

# What is Kubernetes?

Kubernetes (k8s) traces its origin story back to an internal system at Google called Borg and its successor, Omega. Props to whichever Star Trek fan named these. Most people who know any Star Trek will get the Borg reference, but if you get the extended [Omega](https://memory-alpha.fandom.com/wiki/Omega_molecule) reference, that means you suffered past the Voyager episode *Threshold* to get to the Borg episodes in season 4, and for that I’m sorry for both of us. No more Star Trek from here on, back to k8s.

At a high level, k8s solves the problem of assigning computational work packaged in containers, to a pool of compute. Once containerisation became widely available on Linux machines, and containers became the de facto way for developers to package their applications, a new problem arose: how do you efficiently parcel out the many containerised applications Google runs between the pools of compute available? It’s not just about applications in isolation either. Applications communicate with each other, with the internet, with databases, and internal infrastructure services. Hardware and software also fail. A system was needed to conduct this orchestra of containerised applications and the networking interplay between them, to bring them back online when they fail, and to scale up or down the allocation of compute to meet demand. Borg was created to solve these problems at Google. It’s the ideological predecessor to k8s, which was the first serious open-source container orchestrator. 

*Author’s MLOps aside: I think the future of ML Ops moves us away from k8s, but we’re not there yet. It’s not really great for the bursty nature of ML work, and when you get into distributed training, you end up installing a bunch of extra things on the cluster to ensure the scheduler behaves. Some of these don’t play nicely with cluster autoscaler. I think we’ll end up evolving in a direction of: better IR-based serialisation and containerisation for models (and associated pre/post processing), combined with better distributed compute primitives designed specifically for the challenges of ML work. k8s isn’t going anywhere for a while though.*

In industry, you’ll most commonly encounter k8s through a managed k8s service (AWS EKS, or GCP GKE, or Azure AKS. IBM - yes, they have a cloud, I didn’t know this either until I worked there - EKS if you’re working somewhere really odd). 

# One Key Principle, Three Key Concepts 

There’s **one** key principle, and **three** key concepts, you need to take away to start with. 

**Key principle**: k8s is *declarative*. You declare - usually via config files - what you’d like the state of the cluster to be, then the system works to do what’s necessary to move the cluster to that state. If something goes wrong, k8s acts to bring the world back in alignment with your declaration. If you change your declaration, k8s begins the right sequence of object destruction and creation to give life to your wishes with minimum disruption to the cluster. If it fails to do so, it attaches errors to the state of the object where things went wrong. When you’re debugging k8s errors, keep this front of mind. The process is always to ask the cluster to describe the state of the erroneous object, and trace things through from there. If you take this bit of intuition away from this article, and nothing else, you’ll be in a good spot. 

**Key concept 1**: *Nodes*. Nodes are the underlying compute for the cluster. If your company is using a managed k8s service, the nodes will probably be instances of the cloud provider’s compute service (AWS EC2 instances for example).

**Key concept 2**: *Pods*. Pods are k8’s most fundamental abstraction. Many of the other abstractions (like Deployments) are combinations of pods and other k8s objects. A pod is one or more containers, allocated a shared files system and networking interface. Everything needed to run a unit of meaningful computational work. The only thing missing is the compute itself, which is what nodes provide. In this way, k8s achieves separation of the compute layer and the application layer. 

**Key concept 3**: *Scheduling*. k8s ships with a default scheduler, and the scheduler's job is to distribute pods among nodes according to their resources. Since it’s a containerised system, nodes can accept as many pods as their computational resources allow. CPU and memory are the key resource types. For ML work, you’ll usually have a group of nodes with GPUs as well. You can give the scheduler hints and constraints for how you’d like it to distribute pods among nodes. k8s has a very unituitively named push/pull pair of systems called ‘taints and tolerations’ and 'affinity and anti-affinity' for accomplishing all of this, which we won’t get into here. An ML example: for distributed training, we usually want pods to be as co-located as possible to minimise the amount of time they spend communicating with each other over the network. And we want to avoid any nodes without GPUs. We setup the taints and tolerations and affinity interplay between nodes and pods such that this occurs.

The key takeaway: k8s expects you to declare the state of your cluster, and it works to bring/keep the cluster in alignment with that state as much as possible, scheduling pods to nodes as capacity permits. Many pods run on a single node.

# Getting Started

**For this article, we’ve assumed the existence of a k8s cluster already, and that you've followed internal documentation necessary to be able to connect to that cluster. If you're not at this point, there are cloud provider specific resources to get you there. I'll also write a future post covering cluster admin as part of an upcoming article on bootstrapping barebones MLOps in small teams. But for now, that's too much detail for a primer. Subscribe to the mailing list if you're interested.**

All of your interaction with k8s takes place via a command line tool called *kubectl*. It’s pronounced ‘cube control’ (that’s [officially](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.9.md#kubectl) how you say it). Commands are loosely of the form ‘kubectl *verb* *noun* *identifier*’. Components in the cluster listen out for these commands and bring your wishes to life (assuming you have the right permissions: if you hit permission errors, you need to talk to your cluster admins)

[Install](https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/) *kubectl* with `brew install kubectl`

## Preserving Your Sanity

It pays off in saved keystrokes and sanity to do some quality of life improvements before we get started. Add the following aliases to your bash profile. 

```
alias k='kubectl'
alias kgp='kubectl get pods'
```

These are the two most impactful aliases. If you get really serious about k8s, you can take inspiration for other aliases from [here](https://github.com/ahmetb/kubectl-aliases).

The next biggest quality of life tool you’ll want is [kns](https://github.com/blendle/kns) - kubernetes namespace switcher. k8s objects reside in namespaces. Without kns, you’ll end up adding a -n [namespace] flag to every single k8s command (which you will quickly get tired of).

```
brew tap blendle/blendle
brew install kns
```

An optional quality of life tool you might want is ktx, if your organisation has multiple k8s clusters (e.g. a staging and production cluster). ktx lets you easily switch between them with ‘ktx clustername’. Under the hood, both of these tools make changes to your .kube/config file on your behalf, which is where your cluster configuration is stored. If you use the Homebrew steps above, you get ktx as well. 

So far, we’ve setup the tools we need to interact with an existing k8s cluster, and have been over some key concepts/terminology. Let’s do two more things in this primer: create and explore a few of these key objects for ourselves, and create a key commands cheat sheet to refer back to. We'll also cover some objects you're likely to make use of for ML work, like secrets. 

# Exploring Cluster Resources

We're going to introduce two of the most important k8s verb commands you'll learn, `get` and `describe`. Get is useful for a bird's eye summary. Describe is for when you want more details, and is particularly useful for debugging (in combination with the `events` command and `logs`).

## Namespaces

In industry, a typical setup from your platform team would be that you have a namespace created for your team, with all of your team’s resources residing in that namespace. Namespaces provide an isolation mechanism, with the names of resources being unique within a namespace. This way, ML-team-1 and ML-team-2 can both have resources named **perform-feature-making-incantations-from-BI-pipeline-output** without worrying about clashes.

If you don’t specify a namespace when creating or inspecting a k8s resource, k8s places/looks for your resource in the default namespace, which is created when the cluster is created to house these lonely objects. For the sort of work you’ll be doing, always remember the namespace. 

To see all the available namespaces on the cluster, `k get namespaces`.

## Nodes

`k get nodes` will list your clusters available nodes. Nodes aren’t specific to a namespace, since they’re the pool of compute available for all of the computational work running on the cluster. To inspect the nodes available on the cluster, kubectl get nodes. To avoid boring you in a primer, we won’t go any further with nodes here. 

**Intuition builder:** You should be seeing a general pattern emerging here, with the `get` verb. `get` provides us with a summary view of the resources of a particular type. Try running the get command again, but specifying the name of one of the nodes returned. Remember the pattern of `k get [noun] [identifier]`. You'll need to singularise the noun when you add in an identifier. 

## Pods

Pods are a namespaced resource (if a namespace isn't specified when a pod is created, it will reside in the default namespace). We need to direct kns to select the right namespce for your team, If you're in team `opaque-codename`, for instance, and your platform team have created that namespace for you, run `kns opaque-codename` to ensure that all future commands apply to that namespace. Otherwise, you'd have to add `-n opaque-namespace` to your command. 

Using the alias we defined earlier, run `kgp` (we defined `kgp` as `kubectl get pods`). You'll see a tabulated list of pods returned. A typical ML pipeline deployed on a k8s cluster, involves (at least) one pod per step of the pipeline. Each pod in the list will have an attached status field. 

Just like with nodes, you can get information on a particular pod: `k get pod pod-name-xyfyh`

If you see an error in the status field, you can get more detailed infromation by describing that pod `k describe pod pod-name-xyfyh` and looking at the event log towards the bottom of the output. 

## Secrets

Secrets are another really common resource type. Often, you'll have API keys that you need available to pods. Typically, these will be environmental variables when the pod runs. However, in order to populate these environmental variables in the first place, we need a way of storing the secrets securely in the cluster itself. If you're debugging a pipeline, and you're getting API key errors, improper secrets might be the source of the problem. Explore your namespace's secrets like so: 

### Listing secrets

`k get secrets` -> tabulated summary of all secrets

`k get secret secret-name` -> summary of individual secret

## External Resources and Operators

k8s is very extensible. Pods, StatefulSets, ReplicaSets, Deployments, are some of the built in resource types. It allows for Custom Resource Definitions (CRD). A really common CRD you'll encounter in industry is External Secrets Operator. This takes care of syncing a secret from a cloud provider's secret store (AWS SecretsManager, for instance), with a k8s secret. If you update the secret in SecretsManager, ESO syncs the update with the corresponding k8s secret. This is particularly useful for sensitive secrets, that are rotated often. 

k8s lets you use all your usual verbs with external resources: 

`k get externalsecrets` -> view the external secrets in your namespace. 

## Debugging Strategy

There are two high-level ways in which a pipeline will fail: 

- pipeline k8s object deployment failures. In which case, the error will be attached to the k8s object where the object occurred, and the tool you need to reach for is probably `k describe pod erroring-pod` 
- logic failures in your code: in which case, you need to look at the pod logs. `k logs erroring-pod` (note: you don't need to include `pod` in this command, it's implicit)


