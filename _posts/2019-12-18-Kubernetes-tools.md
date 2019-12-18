---
layout: post
title:  "Kubernetes toools for productivity"
author: Ramesh
categories: [ Kubernetes, Tools, Productivity ]
image: assets/images/kubernetes.png
hidden: false
---

## Introduction
Working on Kubernetes can be cumbersome and time consuming especially, when you are working on multiple clusters. I have compiled the below list of tools, which really helped me to enhance my productivity while working on kubernetes.

### Kubectx
Kubectx is a tool to switch between kubernetes contexts. It is written by 'Ahmet Alp Balkan' and he works for Google Cloud as Developer advocate. It is really handy tool especially when you are working with multiple clusters. It can save lot of typing time and helps you quickly switch back and forth between different clusters. Kubectx works seamlessly with different shells like Bash, Zsh, fish etc.

#### Installation
'Kubectx' can be installed on MacOS and different linux distributions like CentOS, AmazonLinux, Ubuntu, Archlinux etc.

There are multiple options to install:
MacOS
  * Homebrew
  * MacPorts
  * Kubectl Plugins
Linux
  * Manual installation by downloading the binary

#### MacOS
  **Homebrew:** If you are using Homebrew, you can install kubectx using below command
```
brew install kubectx
```
  **MacPorts:** If you are using Homebrew, you can install kubectx using below command

```
sudo port install kubectx
```
#### Linux
  Since kubectx is written in bash, it can be run on almost all the linux distributions. You can use the below steps to install it.
```
wget https://github.com/ahmetb/kubectx/blob/master/kubectx
chmod +x kubectx
sudo mv kubectx /usr/local/bin/
```
