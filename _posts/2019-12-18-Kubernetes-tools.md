---
layout: post
title:  "Top 10 Most effective Kubernetes tools to improve productivity"
author: Ramesh
categories: [ Kubernetes, Tools, Productivity ]
image: assets/images/kubernetes.png
hidden: false
---


### 1. Kubectx
Kubectx is a tool to switch between Kubernetes contexts. I work on multiple Kubernetes clusters on daily basis. I generally work on EKS but sometimes I also prefer to test something really quick on minikube. Often it is hard to keep track of which cluster you are currently working. Executing commands without knowing on which cluster you are working can be fatal. Kubectx is really handy when you are working with multiple clusters. It helps me to check and switch b/w different clusters by switching contexts and saves a lot of time. Especially, when there is any production issue going on.

Kubectx works seamlessly with different shells like Bash, Zsh, fish, etc. Personally I prefer Zsh and Fish.

#### Examples

```
$ kubectx minikube
Switched to context "minikube".

$ kubectx -
Switched to context "oregon".

$ kubectx -
Switched to context "minikube".

$ kubectx prod=EKS_us-east-2
Context "prod" set.
Aliased "EKS_us-east-2" as "dublin".
```
### 2. kubens

Kubens is quite similar to kubectx but for namespaces. Kubens is a tool to switch between namespaces.

***kubens*** helps you switch between Kubernetes namespaces smoothly:

#### Examples:
```
$ kubens kube-system
Context "test" set.
Active namespace is "kube-system".

$ kubens -
Context "test" set.
Active namespace is "default".
```
For installation and usage guide please refer to kubens github page.

### 3. Kubetail

Bash script that enables you to aggregate (tail/follow) logs from multiple pods into one stream. This is the same as running "kubectl logs -f " but for multiple pods.

#### Examples:
First, find the names of all your pods:
```
$ kubectl get pods
```
This will return a list looking something like this:
```
NAME                   READY     STATUS    RESTARTS   AGE
app1-v1-aba8y          1/1       Running   0          1d
app1-v1-gc4st          1/1       Running   0          1d
app1-v1-m8acl  	       1/1       Running   0          6d
app1-v1-s20d0  	       1/1       Running   0          1d
app2-v31-9pbpn         1/1       Running   0          1d
app2-v31-q74wg         1/1       Running   0          1d
my-demo-v5-0fa8o       1/1       Running   0          3h
my-demo-v5-yhren       1/1       Running   0          2h
```
To tail the logs of the two "app2" pods in one go simply do:
```
$ kubetail app2
```
To tail only a specific container from multiple pods specify the container like this:
```
$ kubetail app2 -c container1
```
You can repeat -c to tail multiple specific containers:
```
$ kubetail app2 -c container1 -c container2
```
To tail, multiple applications at the same time separate them by a comma:
```
$ kubetail app1,app2
```
For advanced matching you can use regular expressions:
```
$ kubetail "^app1|.*my-demo.*" --regex
```
### 4. kubespy

kubespy is a small tool that makes it easy to observe how Kubernetes resources change in real-time.

Kubespy, can really help you to find answers to below questions:

What happens when you boot up a Pod? What happens to a Service before it is allocated a public IP address? How often is a Deployment status changing?

#### Examples
```
$ kubectl get deploy
NAME READY UP-TO-DATE AVAILABLE AGE
locust-master 1/1 1 1 14d
locust-slave 2/2 2 2 14d
$ kubespy trace deployment locust-slave
[ADDED extensions/v1beta1/Deployment]  default/locust-slave
    Rolling out Deployment revision 1
    ✅ Deployment is currently available
    ❌ Rollout has failed; controller is no longer rolling forward:

ROLLOUT STATUS:
- [Current rollout | Revision 1] [ADDED]  default/locust-slave-7c57b4f6cc
    ✅ ReplicaSet is available [2 Pods available of a 2 minimum]
       - [Ready] locust-slave-7c57b4f6cc-npwz6
       - [Ready] locust-slave-7c57b4f6cc-6hxdh
```
kubespy trace deployment locust-mast will "trace" the complex changes a complex Kubernetes resource makes in the cluster (in this case, a Deployment called locust-slave), and aggregate them into a high-level summary, which is updated in real-time.

kubespy status v1 Pod locust-slave will wait for a Pod called locust-slave to be created, and then continuously emit changes made to its .status field.

### 5. kube-monkey
Kube-monkey is an implementation of [Netflix's Chaos Monkey](https://github.com/Netflix/chaosmonkey) for [Kubernetes](http://kubernetes.io/) clusters. It randomly deletes Kubernetes (k8s) pods in the cluster encouraging and validating the development of failure-resilient services.

Kube-monkey can be really helpful for Disaster Recovery testing, Outage testing etc. If you are running a banking system, e-commerce websites like amazon, online booking etc. It can be a really useful tool to test your application resiliency.

Kube-monkey works in the opt-in model. For example, if want some pods to opt-in for chaos, you have to create the pods with certain labels. You can also whitelist or blacklist namespaces. For example, if you don't want your Kube-system namespace to participate in chaos, you can blacklist it.

For setting up the Kube-monkey in your cluster, you can follow the complete instructions on official [Github](https://github.com/asobti/kube-monkey) repo.

As per my personal recommendation, it is best to start with some examples which are already included in the above mentioned official Github repo. Once you get the confidence, you can start implementing it in your production cluster.

Example of opted-in Deployment killing one pod per purge
```
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: monkey-victim
  namespace: app-namespace
  labels:
    kube-monkey/enabled: enabled
    kube-monkey/identifier: monkey-victim
    kube-monkey/mtbf: '2'
    kube-monkey/kill-mode: "fixed"
    kube-monkey/kill-value: '1'
spec:
  template:
    metadata:
      labels:
        kube-monkey/enabled: enabled
        kube-monkey/identifier: monkey-victim
[... omitted ...]
```
### 6. Kubeapps

Kubeapps is a web-based UI for deploying and managing applications in Kubernetes clusters. It can be very useful when you don't want your Devs or other engineers who are not much familiar with kubectl command line. They can deploy and manage their applications from kubeapps dashboard.

For installation and setup steps, you can follow the instruction from [getting started guide](https://github.com/kubeapps/kubeapps/blob/master/docs/user/getting-started.md) Github page.

### 7. Stern

Stern another tool to tails the pod logs. It allows you to tail multiple pods and multiple containers. The output is color-coded which is really helpful for quicker debugging.

a stern query is a regular expression so the pod name can easily be filtered and you don't need to specify the exact id. If a pod is deleted it gets removed from the tail and if a new pod is added it automatically gets tailed.

You can tail logs of all the pods from a namespace or tail logs for all the pods with a specific label.
Examples:

Tail the staging namespace excluding logs from istio-proxy container
```
stern -n staging --exclude-container istio-proxy
```
View pods from another namespace
```
stern kubernetes-dashboard --namespace kube-system
```
Tail the pods filtered by run=nginx label selector across all namespaces
```
stern --all-namespaces -l run=nginx
```
Follow the frontend pods in canary release
```
stern frontend --selector release=canary
```
Pipe the log message to jq:
```
stern backend -o json | jq .
```
Only output the log message itself:
```
stern backend -o raw
```
stern is really useful for troubleshooting. It saves a lot of time as logs are color-coded and you can review the logs pretty quicky from the whole cluster, instead of struggling to find the exact kubectl filter commands to locate specific pod to view the logs.

### 8. Kube-bench

Kube-bench is a Go application that checks your Kubernetes cluster whether it is deployed securely by running the checks documented in the [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes/).

There are different test suites for master and worker nodes, and for nodes in federated deployments.

It is really a great tool for the enterprises to check their setup against CIS benchmarking. Irrespective of your company size, you can also use this tool to test your cluster setup for security loopholes. It really provides a detailed report about the issues and recommendations.

Note that it is impossible to inspect the master nodes of managed clusters, e.g. GKE, EKS, and AKS, using Kube-bench as one does not have access to such nodes, although it is still possible to use Kube-bench to check worker node configuration in these environments.

Kube-bench can be run as a docker container from your local setup or inside the Kubernetes cluster as a pod.

Tests are configured with YAML files, making this tool easy to update as test specifications evolve.

### 9. kubefwd (Kube Forward)

kubefwd is a command-line utility built to port forward multiple services within one or more namespaces on one or more Kubernetes clusters. kubefwd uses the same port exposed by the service and forwards it from a loopback IP address on your local workstation. kubefwd temporally adds domain entries to your /etc/hosts file with the service names it forwards.

It is yet another great Kubernetes tool from the community which really helped our Dev and Ops team to test their applications fasters and you don't have to worry about making all those changes manually to run a Kube-proxy locally and map different services on different ports. kubefwd takes care of that for you. It really helps us in saving a lot of time and improving productivity while working on Kubernetes.

When working on our local workstation, my team and I often build applications that access services through their service names and ports within a Kubernetes namespace. kubefwd allows us to develop locally with services available as they would be in the cluster.

For installation and usage guide can be found [here](https://imti.co/kubernetes-port-forwarding/).

### 10. kube-shell

Kube-shell is another my favorite tool that really provide a lot of features like auto-completion, search, suggestions, flags descriptions etc. for kubectl command line. It is an integrated shell for working with the kubectl CLI. It helps you save a lot of typing especially if you like to use kubectl CLI command. You don't have to remember all the kubectl options or flags.

If you are a [zsh](https://github.com/ohmyzsh/ohmyzsh) or [fish](https://github.com/fish-shell/fish-shell) shell user you would love using kube-shell as it gives a similar experience.

For installation and detailed usage guide you can follow the instructions mentioned on their official [GitHub](https://github.com/cloudnativelabs/kube-shell) page.
