
# k8smaker
A fully scripted Kubernetes deployment

This is a personal project to make it easier to setup a Kubernetes cluster from scratch, fully ready to launch pods.  It's a fully scripted deployment by way of a couple of Bash scripts that simply and cleanly arranges the prerequisites for K8s, allow for an explicit data drive where `etcd` and `Docker` data are stored.  If you have read many walkthroughs on installing Kubernetes, these are pretty much all those steps and suggestions, smashed into a single script and automated.  Watch this space, as this is very early and under rapid development.

All these scripts are intended to run from the first machine in the k8s cluster, which we'll call `control-node1`.  It sets up an ssh key on initialization and copies it to the other nodes so it can poke in the appropriate script and install.

## Cluster Configuration

 - Kubernetes 1.18.4
 - Ubuntu 18.04LTS
 - Bare metal or AWS EC2
 - etcd and control plane are stacked on control nodes
 -- A smaller box will suffice to manage the cluster
 -- Setup for HA control cluster, even if you only have one to start with
 - pods run on worker nodes
 -- Beefier boxes should be used for these
 - **Calico** for networking automatically configured to talk to secured etcd
 - **Istio** for ingress and mesh routing

## Some parts run as Root
Tested on Ubuntu 18.04LTS only.  May run on any reasonably vanilla `systemd` based Debian-based Linux.  Do not run these scripts on anything except a newly installed machine.  **Data loss is very possible.**  As with anything you download off the internet, running a script as `root` is dangerous and until it's proven trustworthy, it may destroy anything or everything it touches.  I promise this script doesn't intend to do that, but why should  you trust me?  **Read it over first, and if you see anything you are concerned with, don't run it!**  Even better, fix it and send me a pull request.

## Prerequisites for k8smaker
All nodes need to be a `systemd`-based Ubuntu 18.04LTS (or similar Debian distro). with OpenSSH installed.  If you missed OpenSSH on the install, do [this walkthrough](https://linuxize.com/post/how-to-enable-ssh-on-ubuntu-18-04/) real quick.  All the nodes need to have the same username, which you can configure in `k8smaker_config`.

## k8smaker_control
This installs updates apt, installs a few packages it needs to do the install like curl, Docker, Kubeadm, Kubectl, and Kubelet.  
