---
layout: single
title: "Day-3: Practical Understanding of Kubernetes Nodes, Workloads, and the Foundations"
description: "A clear introduction to Kubernetes nodes and workloads, with a practical look at how these core components shape effective security testing from the ground up."
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
  name: "Day 3: Practical Understanding of Kubernetes Nodes, Workloads, and the Foundations"
  headline: "Day 3: Practical Understanding of Kubernetes Nodes, Workloads, and the Foundations"
  description: "Understand Kubernetes nodes and workloads, and learn how these core building blocks form the foundation for practical and effective Kubernetes security testing."
  keywords: "Kubernetes, Kubernetes Nodes, Kubernetes Workloads, Kubernetes Security, Container Security, DevOps"
---

It’s been a while since I last posted, but I’m back with Day 3 of learning Kubernetes.

In this post, I’m shifting focus towards the DevOps side of Kubernetes. From a security testing perspective, it’s hard to identify real vulnerabilities unless we understand how Kubernetes actually works in practice. Security issues rarely exist in isolation. Most of them come from the way things are deployed, configured, and operated.

To carry out meaningful Kubernetes security assessments and suggest practical remediations, we don’t need to become full-time DevOps engineers. That said, solid foundational and hands-on knowledge is essential. Understanding how workloads run, where configurations live, and how different components interact helps us identify the kinds of misconfigurations attackers exploit in real environments.

This blog focuses on the practical workings of Kubernetes rather than theory-heavy explanations. We’ll build on these concepts step by step in the upcoming posts.

### Nodes

Nodes are the worker machines in a Kubernetes cluster. This is where your applications actually run.

Each node provides:

- CPU and memory to run workloads  
- Networking for pod-to-pod and service communication  
- A runtime environment managed by Kubernetes  

In simple terms, pods do not run directly on the cluster. They always run on nodes. If there are no worker nodes available, workloads cannot be scheduled.

## Checking Nodes in a Cluster

A common first step when interacting with any Kubernetes cluster is to understand how many nodes exist and their current state.

To list all nodes with extended details:

```bash
kubectl get nodes -o wide
```

This shows useful information such as internal IPs, operating system, and container runtime.

For a simpler view:
```bash
kubectl get nodes
```

To explicitly check the kubelet version running on each node:
```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,KUBELET_VERSION:.status.nodeInfo.kubeletVersion
```

This is especially useful during security reviews, as outdated kubelet versions can introduce known vulnerabilities.

For detailed information about a specific node:

```bash
kubectl describe node <node-name>
```
This command reveals:
- Node conditions
- Resource capacity and allocation
- Running pods
- Labels and taints

<img width="700" height="500" alt="image" src="https://github.com/user-attachments/assets/5918562a-7cb4-4922-8cb8-0c0cd0ea09c1" />


From a security perspective, this data is extremely valuable. Misconfigured nodes, excessive privileges, or exposed resources often become the entry point for attackers.

## Kubernetes Version and Node Operating System Details

After identifying how many nodes exist, the next important step is understanding what versions are actually running underneath. From a security perspective, this matters more than most people realise. Vulnerabilities often target specific Kubernetes, kubelet, or kernel versions rather than the application itself.


#### Checking the Kubernetes Version on Nodes

While the control plane defines the cluster version, each node runs its own kubelet. If kubelet versions drift or become outdated, they can introduce security gaps.

To check Kubernetes-related version details per node, you can inspect node information directly.

#### Identifying the Operating System and Kernel

Kubernetes runs on top of an operating system, and that operating system is part of the attack surface. Kernel vulnerabilities, outdated distributions, or unsupported architectures can all be leveraged during real-world attacks.

To extract operating system, kernel, and architecture details for all nodes:

```bash
kubectl get nodes -o json | jq -r '.items[].status.nodeInfo | "\(.machineID) \(.osImage) \(.kernelVersion) \(.architecture)"'
```

This command provides a quick overview of:
- OS distribution and version
- Kernel version
- CPU architecture

For targeted inspection of a specific node, you can use:
```bash
kubectl describe node <node-name> | grep -i "OS Image"
kubectl describe node <node-name> | grep -i "Kernel Version"
```

If you prefer a cleaner, table-style output across all nodes:
```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,OS_IMAGE:.status.nodeInfo.osImage,KERNEL:.status.nodeInfo.kernelVersion,ARCH:.status.nodeInfo.architecture
```
<img width="700" height="500" alt="image" src="https://github.com/user-attachments/assets/1bddca59-f97b-4636-afb4-a9d2c3c63086" />

##### Why This Matters for Security Testing:

From a Kubernetes security testing perspective, this information is critical:
- Outdated kernels may expose privilege escalation vulnerabilities
- Unsupported OS images often miss security patches
- Mixed node versions can indicate weak operational hygiene
- Attackers frequently fingerprint node environments before exploitation

Understanding node operating systems and Kubernetes versions helps map known CVEs to real infrastructure instead of guessing. This is a foundational step before moving deeper into pod security, container runtimes, and workload isolation.


### Pods

A Pod is the smallest deployable unit in Kubernetes. It represents one or more tightly coupled containers that:

- Share the same network namespace  
- Share storage volumes  
- Are scheduled and managed together  

From a security point of view, this matters because containers inside a pod trust each other by default. If one container is compromised, the others in the same pod are usually affected as well.

#### Checking How Many Pods Exist

To see all pods running across the entire cluster:

```bash
kubectl get pods --all-namespaces
```

If you only want the total number of pods:
```bash
kubectl get pods --all-namespaces --no-headers | wc -l
```

To count pods in the current default namespace:
```bash
kubectl get pods --no-headers | wc -l
```
<img width="700" height="500" alt="image" src="https://github.com/user-attachments/assets/18025dbd-4441-4811-a42a-7c0da5d785bc" />


Understanding pod count is useful during security reviews. A high number of unexpected pods can indicate misconfigurations, leftover test workloads, or even compromised deployments.

#### Creating a New Pod

Let’s create a simple pod using the nginx image:
```bash
kubectl run nginx-pod --image=nginx
```

After creating the pod, verify that it is running:
```bash
kubectl get pods
```
<img width="700" height="500" alt="image" src="https://github.com/user-attachments/assets/f26c591a-277b-4414-813d-65ad50b0ea2b" />


Now check how many pods exist in the default namespace:
```bash
kubectl get pods --no-headers | wc -l
```

You can also list them to visually confirm:
```bash
kubectl get pods
```
<img width="700" height="500" alt="image" src="https://github.com/user-attachments/assets/906598f1-bc7c-412a-8987-30468332e26e" />


#### Pod Health and Restarts

If you notice a high restart count for a pod, it usually means something is wrong. Common causes include:
- Application crashes
- Resource limits being hit
- Misconfigured liveness or readiness probes
- Permission or configuration issues

Ignoring frequent restarts often leads to unstable workloads and hidden security risks.

#### Inspecting Pod Details

To get detailed information about a pod, including its configuration, current state, and events:
```bash
kubectl describe pod <pod-name>
```

For example:
```bash
kubectl describe pod nginx
```
<img width="700" height="500" alt="image" src="https://github.com/user-attachments/assets/38947358-c564-4c38-9d78-ad0b67e1c940" />


The Events section at the bottom of this output is especially valuable. It often reveals:
- Image pull failures
- Crash loops
- Scheduling problems
- Permission or volume mount errors
  <img width="700" height="500" alt="image" src="https://github.com/user-attachments/assets/2d08acab-0779-4904-af3d-48bd688d1eea" />


From a security testing perspective, these events frequently expose misconfigurations that attackers may exploit.

#### Deleting Pods

Pods can be deleted either using a manifest file or directly by name.

Using a manifest file:
```bash
kubectl delete -f <manifest-file>
```

Example:
```bash
kubectl delete -f podmanifest.yml
```
<img width="488" height="72" alt="image" src="https://github.com/user-attachments/assets/1acb2a90-50fb-4249-8841-2886d3024d70" />


Using the pod name:
```bash
kubectl delete pod nginx
```
<img width="398" height="75" alt="image" src="https://github.com/user-attachments/assets/5fdb8ae6-3b2f-4650-a3d5-66789ca13718" />


Deleting pods is a normal part of Kubernetes operations. During security assessments, however, unexpected pod deletions or constant recreation can indicate unstable deployments or malicious activity.


### Namespaces

A namespace is a logical partition within a Kubernetes cluster. It allows you to group and isolate resources such as pods, services, and configurations while still running on the same underlying cluster.

Namespaces are heavily used in real-world environments for:

- Environment separation like dev, staging, and production
- Multi-team or multi-tenant isolation
- Applying security controls and resource limits

From a security perspective, namespaces are often mistaken for strong isolation. In reality, they are organisational boundaries unless combined with proper RBAC, network policies, and admission controls.

#### Listing Namespaces

To list all namespaces in the cluster:
```bash
kubectl get namespace
```
<img width="366" height="149" alt="image" src="https://github.com/user-attachments/assets/715b966a-afa0-4e7c-994d-d1c3eeb38656" />


This gives a quick overview of how resources are logically separated inside the cluster.

#### Viewing Pods in a Specific Namespace

To list all pods running inside a particular namespace:
```bash
kubectl get pods -n <namespace-name>
```

For example, to view pods in the kube-system namespace:
```bash
kubectl get pods -n kube-system
```
<img width="727" height="205" alt="image" src="https://github.com/user-attachments/assets/ad30888e-2ee3-4115-8d1d-067303630189" />


The kube-system namespace contains core Kubernetes components and system services. From a security testing angle, this namespace is especially sensitive and should be tightly restricted.

#### Creating a New Namespace

To create a new namespace:
```bash
kubectl create ns <namespace-name>
```

Example:
```bash
kubectl create ns demo-singh
```
<img width="440" height="72" alt="image" src="https://github.com/user-attachments/assets/7f12f842-440a-4253-8a8e-8cc56582b1a3" />


Namespaces like this are commonly used to separate applications or teams and apply scoped permissions.

#### Creating Pods Inside a Specific Namespace

To ensure a pod is created inside a particular namespace, you must explicitly define the namespace under the metadata section of the manifest file.

Example pod manifest:
```bash
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: demo-singh
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

Apply the manifest:
```bash
kubectl apply -f podmanifest.yml
```

#### Verifying Pods in the Namespace

To see pods running under a specific namespace:
```bash
kubectl get pod -n <namespace-name>
```

Example:
```bash
kubectl get pod -n demo-singh
```
<img width="467" height="188" alt="image" src="https://github.com/user-attachments/assets/de49bf16-236f-4fb0-bc60-df4120466750" />


#### Security Note on Namespaces

Namespaces alone do not prevent lateral movement. If RBAC rules are too permissive or network policies are missing, an attacker who gains access to one namespace may still move across others.

Understanding namespaces is a key step before exploring:

- Role-Based Access Control
- Network Policies
- Resource quotas
- Pod security controls


### ResourceQuota Configuration

A ResourceQuota is used to control how much CPU and memory can be consumed within a namespace. It prevents workloads from using more resources than intended and helps protect the cluster from noisy neighbours or accidental resource exhaustion.

In real environments, missing or weak quotas are a common operational and security issue. An attacker or a misconfigured application can easily cause denial-of-service conditions if resource usage is not restricted.

#### Defining a ResourceQuota

The following ResourceQuota limits the total CPU and memory usage in a namespace to:

- 1 CPU core
- 1 GiB of memory
  
```bash
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tiny-rq
spec:
  hard:
    cpu: "1"
    memory: 1Gi
```

This configuration ensures that all pods combined within the namespace cannot exceed these limits.

## Applying the ResourceQuota to a Namespace

To apply the ResourceQuota to a specific namespace:
```bash
kubectl apply -f resouce_quota.yaml -n <namespace-name>
```

Example:
```bash
kubectl apply -f resouce_quota.yaml -n demo-singh
```

Once applied, Kubernetes enforces these limits at the namespace level.

#### Verifying ResourceQuota on a Namespace

To view detailed information about a namespace, including any resource quotas applied to it:
```bash
kubectl describe ns <namespace-name>
```
Example:
```bash
kubectl describe ns demo-singh
```
<img width="625" height="413" alt="image" src="https://github.com/user-attachments/assets/5923f1ab-7b72-417d-8e5e-c32881d59b52" />


This output shows:

- Namespace status
- Labels and annotations
- Active resource quotas and their usage

## Why ResourceQuotas Matter for Security

From a security testing perspective, ResourceQuotas:

- Reduce the impact of compromised workloads
- Prevent cluster-wide resource exhaustion
- Enforce basic operational hygiene
- Limit damage caused by runaway pods or malicious containers

ResourceQuotas are not a complete security control, but they form an important defensive layer when combined with proper limits, RBAC, and monitoring.



### Resource Monitoring

Resource monitoring is a critical part of understanding how a Kubernetes cluster behaves in real time. Without visibility into CPU and memory usage, it is difficult to detect performance issues, misconfigurations, or early signs of abuse.

Kubernetes relies on the Metrics Server to expose basic resource usage data for nodes and pods.

#### Installing the Metrics Server

To install the Kubernetes Metrics Server using the latest official release manifest:
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Give it a minute or two to spin up after applying the manifest.

#### Verifying Metrics API Availability

Before running any resource monitoring commands, confirm that the Metrics API is available:
```bash
kubectl get apiservices | grep metrics
```
<img width="764" height="74" alt="image" src="https://github.com/user-attachments/assets/ad376fe7-466c-469e-9027-fba9b54fa7d6" />


If the service is listed and available, you are good to proceed. If not, wait a short while and check again.

#### Viewing Node Resource Usage

To see CPU and memory usage across all nodes in the cluster:
```bash
kubectl top nodes
```
<img width="554" height="92" alt="image" src="https://github.com/user-attachments/assets/dba443f6-68df-4d71-ad67-577550d6d9b7" />


This provides a quick snapshot of how heavily each node is being utilised and helps identify overloaded or underused nodes.

#### Viewing Pod Resource Usage

To view resource usage for all pods across every namespace:
```bash
kubectl top pods -A
```

Or equivalently:
```bash
kubectl top pods --all-namespaces
```
<img width="764" height="385" alt="image" src="https://github.com/user-attachments/assets/ad7a72d9-60d9-4a12-bccc-45fe615bd602" />


This is especially useful when:

- Debugging performance issues
- Validating resource limits and quotas
- Identifying noisy or misbehaving pods

#### Why Resource Monitoring Matters for Security

From a security testing perspective, resource metrics can reveal:

- Crypto-mining or abuse workloads
- Pods consuming unexpected levels of CPU or memory
- Denial-of-service conditions in early stages
- Poorly defined limits that attackers could exploit

Resource monitoring does not replace logging or runtime security, but it provides essential operational visibility that often highlights problems before they escalate.



### Requests and Limits

Requests and limits define how much CPU and memory a container is guaranteed and how much it is allowed to consume at most. These settings directly affect scheduling, performance, and overall cluster stability.

In many real-world security incidents, missing or incorrect limits have allowed a single pod to degrade or crash an entire node.

#### Defining Requests and Limits for a Pod

Below is an example of a pod running NGINX with explicit CPU and memory requests and limits:
```bash
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
spec:
  containers:
    - name: nginx
      image: nginx:1.14.2
      resources:
        requests:
          cpu: 250m
          memory: "65M"
        limits:
          cpu: 500m
          memory: "130M"
```

In this configuration:

- Kubernetes guarantees the requested resources during scheduling
- The container is prevented from exceeding the defined limits

#### Describing a Pod in a Namespace

To inspect a pod and confirm its requests, limits, and runtime state:
```bash
kubectl describe pod -n <namespace>
```

Example:
```bash
kubectl describe pod -n demo-singh
```
<img width="548" height="643" alt="image" src="https://github.com/user-attachments/assets/d0977d1a-14c0-4592-8ad4-482e5a1ef315" />

This output shows:

- Resource requests and limits
- Current pod status
- Events and any resource-related errors

#### Understanding Requests vs Limits

Requests define the minimum resources a container is guaranteed.

Limits define the hard maximum the container cannot exceed.

If a container exceeds its memory limit, it is killed and restarted.  
If it exceeds its CPU limit, it is throttled.

#### Why This Matters for Security

From a security testing and hardening perspective:

- Pods without limits can exhaust node resources
- Compromised containers can be abused for activities like crypto-mining
- Resource starvation can cause denial-of-service for other workloads

Requests and limits are not just performance controls. They are an important defensive measure that limits the blast radius of compromised or misbehaving containers.



### Probes

Probes are health checks that tell Kubernetes whether a container is running correctly, ready to receive traffic, or has started properly. They play a key role in application stability and recovery.

Poorly configured or missing probes are a common cause of unstable workloads and can hide real issues during security assessments.

#### Types of Probes

Liveness probe  
Restarts the container if it becomes unhealthy

Readiness probe  
Removes the pod from service traffic when it is not ready

Startup probe  
Prevents slow-starting applications from being restarted too early

Each probe serves a different purpose and they are often used together in production environments.

#### Example: Liveness Probe Configuration

Below is an example of a pod configured with a liveness probe:
```bash
apiVersion: v1
kind: Pod
metadata:
  name: sise-lp
spec:
  containers:
    - name: sise
      image: mhausenblas/simpleservice:0.5.0
      ports:
        - containerPort: 9876
      livenessProbe:
        initialDelaySeconds: 2
        periodSeconds: 5
        timeoutSeconds: 1
        failureThreshold: 3
        httpGet:
          path: /health
          port: 9876
```

This configuration means:

- Kubernetes waits 2 seconds before performing the first health check
- Health checks run every 5 seconds
- Each check times out after 1 second
- After 3 consecutive failures, the container is restarted

#### Why Probes Matter for Security

From a security perspective, probes help surface real problems instead of hiding them:

- Crashing or unstable containers can indicate exploitation attempts
- Missing probes can allow broken services to keep receiving traffic
- Restart loops may signal resource abuse or misconfiguration
- Probes improve observability during incident response

Probes are not a security control by themselves, but they provide important signals that help identify abnormal behaviour early.


This post covered the practical side of Kubernetes from a developer and operational perspective. Understanding how nodes, pods, namespaces, resource management, and health checks work is essential before diving deeper into security testing.

In the upcoming posts, we’ll explore Kubernetes in more depth and gradually move into real-world security issues, common misconfigurations, and practical remediation techniques.

In the meantime, if you have any questions or would like to connect, feel free to reach out to me on LinkedIn. I’m always happy to connect.

Thanks for reading, and see you in the next post.
