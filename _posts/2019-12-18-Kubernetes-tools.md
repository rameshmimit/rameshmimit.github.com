---
layout: post
title:  "Kubernetes toools for productivity"
author: Ramesh
categories: [ Kubernetes, Tools, Productivity ]
image: assets/images/kubernetes.png
hidden: false
---

#### Introduction
Working on Kubernetes can be cumbersome and time consuming specially, when you are working on multiple cluster. I have compiled a list of couple of tools, which can really help you to enhance your productivity while working on kubernetes.

#### Kubectx
Kubectx is a tool to switch between kubernetes contexts. It is written by 'Ahmet Alp Balkan' and he works for Google Cloud as Developer advocate. It is really handy tool especially when you are working with multiple clusters. Kubectx works seamlessly with different shells like Bash, Zsh, fish etc.

### Installation
'Kubectx' can be installed on MacOS and different linux distributions like CentOS, AmazonLinux, Ubuntu, Archlinux etc.

There are multiple options to install:
MacOS
  * Homebrew
  * MacPorts
  * Kubectl Plugins
Linux
  * Manual installation by downloading the binary

### MacOS
  **Homebrew:** If you are using Homebrew, you can install kubectx using below command
```
brew install kubectx
```
  **MacPorts:** If you are using Homebrew, you can install kubectx using below command

```
sudo port install kubectx
```
### Linux
  Since kubectx is written in bash, it can be run on almost all the linux distributions. You can use the below steps to install it.
```
Download the kubectx binary from https://github.com/ahmetb/kubectx/blob/master/kubectx
chmod +x kubectx
sudo mv kubectx /usr/local/bin/
```
