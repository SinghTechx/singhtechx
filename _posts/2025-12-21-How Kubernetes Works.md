---
layout: single
title: "Day-2: Understanding How Kubernetes Works"
description: "Before diving into Kubernetes security issues, it is essential to understand how Kubernetes works from a developer’s perspective. This post explains Kubernetes architecture, workflow, and core components in a simple and practical way."
excerpt_separator: "<!--more-->"
classes: wide
categories:
  - Kubernetes Journey
tags:
  - kubernetes
  - k8s
  - kubernetes security
  - container security
  - devops security
  - cloud native
seo:
  type: BlogPosting
  name: "How Kubernetes Works: A Developer’s Point of View"
  headline: "How Kubernetes Works: A Developer’s Point of View"
  description: "Learn how Kubernetes works from a developer’s point of view before diving into real-world Kubernetes security issues and remediation."
  keywords: "Kubernetes, Kubernetes Architecture, Kubernetes Security, DevOps, Containers"
---

Before diving into security concerns in Kubernetes, it is important to understand how Kubernetes actually works from a developer’s point of view. Security issues and their remediation only make real sense when you know what is happening behind the scenes and how developers interact with the platform day to day.

This blog focuses on building a clear mental model of Kubernetes rather than jumping straight into commands or security controls.

---

### The Problem Kubernetes Is Trying to Solve

Modern applications are no longer single servers running a single process. Developers build applications as multiple services, each running in containers. These containers need to be deployed, scaled, restarted when they fail, and connected to each other.

Doing this manually becomes painful very quickly.

Kubernetes exists to solve this operational problem. It takes care of running containers reliably, at scale, across many machines, while giving developers a consistent way to deploy and manage applications.

---

### Kubernetes at a High Level

In simple terms, Kubernetes is a system that runs and manages your containers for you.
Instead of manually starting, stopping, or fixing applications, you just tell Kubernetes how your application should run, for example how many copies you need and which container image to use.

Kubernetes then keeps checking the system and automatically fixes things if something goes wrong, making sure your application always stays in the state you asked for.

From a developer’s perspective, you are not managing servers. You are describing applications.

Two simple ideas are important to understand:

Declarative configuration: you tell Kubernetes what you want your application to look like, for example how many copies should be running. You do not tell it the step by step process to achieve that.

Continuous reconciliation: Kubernetes keeps watching the cluster all the time. If something changes or breaks and the current state no longer matches what you asked for, Kubernetes automatically tries to fix it.

---

### The Cluster Concept

A Kubernetes setup is called a cluster. A cluster is made up of two main parts:

- Control Plane  
- Worker Nodes  

As a developer, you mostly interact with the control plane, even though your application actually runs on the worker nodes.

---
<img width="700" height="500" alt="image" src="https://github.com/user-attachments/assets/18d035ec-647b-4d2a-8d97-56ada969e6da" />

*Image credit: Kubesphere*

### Control Plane – The Brain of Kubernetes

The control plane makes all the decisions. It does not run your application containers directly, but it decides where and how they should run.

#### API Server

The API server is the front door of Kubernetes. Every request goes through it, whether it comes from `kubectl`, a CI pipeline, or another Kubernetes component.

From a security perspective, this is critical because authentication, authorisation, and admission controls all happen here.

#### Scheduler

The scheduler decides which worker node should run a new Pod. It looks at available resources, constraints, and policies before making a decision.

#### Controller Manager

Controllers continuously monitor the cluster. If something is missing or broken, they try to fix it. For example, if you declare three replicas and one Pod crashes, the controller creates a new one automatically.

#### etcd

etcd is the key-value store that holds the entire state of the cluster. Everything Kubernetes knows about workloads, configuration, and secrets lives here.

This is one of the most sensitive components from a security testing point of view.

---

### Worker Nodes – Where Applications Run

Worker nodes are the machines that actually run your containers.

Each node includes:

#### Kubelet

The kubelet communicates with the control plane and ensures the containers defined in Pod specifications are running correctly.

#### Container Runtime

This is the software that runs containers, such as containerd or CRI-O.

#### Kube Proxy

Kube proxy manages networking rules so Pods and Services can communicate reliably across the cluster.

---

### Pods – The Smallest Deployable Unit
A Pod is the most basic thing you can create in Kubernetes. It is the unit that Kubernetes actually runs. You do not run containers by themselves. Kubernetes always runs them inside Pods.

Most of the time, a Pod has just one container. Sometimes, a Pod can have more than one container when those containers need to work very closely together and share the same network and storage.

As a beginner, it helps to remember this:
You do not deploy containers directly in Kubernetes. You deploy Pods, or other resources like Deployments that automatically create and manage Pods for you.

---

### Deployments and Desired State

A Deployment is the most common way developers run applications in Kubernetes.

You define:

- The container image  
- The number of replicas  
- The update strategy  

Kubernetes then ensures the desired number of Pods is always running. If a Pod crashes or a node goes down, Kubernetes replaces it automatically.

This level of abstraction is very useful because it makes life easier for developers, but it can also hide what is really happening in the background. From a security point of view, it is important to know exactly what Kubernetes is creating and managing for you, otherwise important risks can be missed.

---

### Services and Networking

Pods are temporary. They can be created and destroyed at any time, so they cannot be relied on directly for networking.

A Service provides a stable endpoint and routes traffic to the correct Pods. This allows applications to communicate without worrying about changing IP addresses.

Misconfigured Services, ingress resources, and missing network policies are common real-world security issues.

---

### Why This Matters for Security

When testing Kubernetes security, you are not just attacking containers. You are testing the control plane, configuration objects, and trust boundaries between components.

---

### Closing Thoughts

Kubernetes is powerful because it takes care of servers and infrastructure details so developers do not have to think about them. However, this can also make security issues harder to notice if you do not understand what Kubernetes is doing behind the scenes.

In upcoming posts, I will build on this foundation and connect real-world Kubernetes security issues to these core components, followed by practical remediation steps using examples from the Kubernetes Goat project.

If you are serious about Kubernetes security testing, this understanding is the starting point.
