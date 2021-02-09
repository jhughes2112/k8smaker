
# k8smaker
Automating all the things!

This is a personal project to make it easier to stand up a Kubernetes cluster from a fresh install of Ubuntu.  The number of times I have done these dozens of steps, I cannot count.  Rather than keep the cheat sheet of commands with me, I wrote this set of scripts so I can essentially wipe out a machine, pull down this repo, and launch a couple of scripts and have a cluster running.  Most importantly, reverse the process and get back to a machine with nothing preventing the re-installation of a cluster.  Easy!

Furthermore, the documentation for all the things you should do to setup a machine for Kubernetes is fairly scattered and hard-won.  Every step is important, of course, but if you have read some of the walkthroughs on installing Kubernetes, most of them include relatively few of the preconditioning steps.  This gathers everything into one place and does all the things that I've seen recommended at each step, in the right order.  All these scripts are intended to run from the first machine in the k8s cluster, which we'll call `control-node`.

These scripts are built to work fine on baremetal or aws, and have slight differences in them to accommodate AWS's quirks.

# Quickstart
## Init a cluster
- `scp ~/.ssh/PRIVATE_KEY_NAME ubuntu@CONTROL_NODE/.ssh/id_rsa` You must be able to ssh from the control node to each worker node, so copy up your private key to the control node.
- `ssh CONTROL_NODE`
- `chmod 0600 ~/.ssh/id_rsa` sets the permissions for your private key to access other worker nodes.  At this point, you could test by ssh'ing to one of your workers from the control node.  Just remember to return to the control node before doing the next step.
- `git clone https://github.com/jhughes2112/k8smaker.git`
- `cd k8smaker`
- Edit **k8smaker_config** and set cluster name, username you will use for ssh (often **ubuntu**), change the bootstrap token, change the control node name to a load balancer DNS entry if you plan to use one, etc.
- `./k8smaker_init_aws` will fully configure and set up a Kubernetes cluster on the local machine with etcd,control,worker roles, but requires some permissions setup (see below)

## For each worker node
- `ssh CONTROL_NODE`
- `cd k8smaker`
- `./k8smaker_addworker_aws ip-172-16-1-144.ec2.internal` will construct a script to run on the specified host, copy it over then run it for you.  All scripts are stored both on the control-node in your home folder as well as on the remote host it was executed on.

## Cluster Configuration
 - Bare metal or AWS. Easy to add more cloud hosts.
 -- Ubuntu 20.04LTS
 -- **Kubernetes** 1.20.1
 -- **Istio** 1.7.1 for ingress and mesh routing
 -- **Calico** 3.17.1 for networking automatically configured to talk to secured etcd
 -- These versions can be changed in one place in the config file, but once you deploy a cluster, don't change them.
 - etcd and control plane are stacked on control nodes
 -- A smaller machine will suffice to manage the cluster (2+CPU, 2+GB RAM)
 -- Setup for High Availability control cluster, even if you only have one to start with
 - Pods run on worker nodes
 -- Use beefy machines for these
 - On AWS, adding a worker also generates an NLB that automatically adds worker nodes when joined to the cluster.

## Prerequisites for k8smaker
 - All nodes need to be a `systemd`-based Ubuntu 18.04LTS (or similar Debian distro)
 - All nodes need [OpenSSH](https://linuxize.com/post/how-to-enable-ssh-on-ubuntu-18-04/) installed, but don't worry about setting up keys.  That's built in.
 - All nodes need the same username (often `ubuntu` but can be anything), and needs `sudo` access.
 - All scripts are to be executed on the `control-node` machine.

## AWS
Because there is a plugin called a "cloud provider" for AWS that knows how to automatically create load balancers for you, and tag resources as they are created so they know how to remove them later, you have to add some permissions as IAM roles for the `control-node` and a more limited set of permissions for the `worker` nodes.
- Go to your AWS console and open the IAM service.
-- Go to the left side and click Policies.  Create Policy.  Click JSON.
-- Copy the control-plane policy from this webpage https://kubernetes.github.io/cloud-provider-aws/prerequisites.html and replace the JSON contents with it.  Click Review Policy.
-- Name it k8s-control-node.  Click Create Policy.
-- Create Policy.  Click JSON
-- Copy the node policy from this webpage https://kubernetes.github.io/cloud-provider-aws/prerequisites.html and replace the JSON contents with it.  Click Review Policy.
-- Name it k8s-worker.  Click Create Policy.
-- Now you have two policies, go back to the IAM dashboard and click Roles on the left.  Create Role.
-- Leave AWS Service selected, and click EC2.  Click Next: Permissions.  
-- Type k8s-control-node in the search box and select it.  Next: Tags.  Next: Review
-- Name the role k8s-control-node.
-- Repeat this process to create a role with k8s-worker.
- Go to the EC2 service.
-- Associate the k8s-contol-node IAM role with your initial instance (either when creating it, or after the fact, but before trying to run k8s_init_aws)
-- You can assign this after creating your instance by clicking EC2 -> Instances -> right click your instance and go to Security -> Modify IAM Role, then select k8s-control-node from the drop down.
-- Any worker nodes can be created with the k8s-worker IAM role.
- Go to the Security Groups and add a group.
-- You'll probably want port 80, 443 open to everyone, but 22 (ssh) and 6443 (kubectl) locked down to just your IP address.
- Tagging Resources
-- The following resources need a tag added in the AWS dashboard, where <CLUSTERNAME> is what you named the cluster in k8s_config.  "kubernetes.io/cluster/CLUSTERNAME" = "owned"
-- Under EC2, all Instances must have this tag.
-- Under EC2, the Security Group should have this tag (so I have read, seems to work w/o this)
-- Under VPC, all Subnets that should carry traffic for you need it (so I have read, seems to work w/o this)
-- Under VPC, the Route Table needs a tag (so I have read, seems to work w/o this)
- Finally, note you can use a storage class called 'gp2' which allocates EBS volumes dynamically.

## k8smaker_config
Edit this file to set your cluster name, username, etc.  This is shared across all the scripts that need it.  See comments for what each variable means.

In my case, I run on bare metal that boots from a slow drive, but have fast data drives.  So this script allows me to configure the device to mount at the data folder location.  If this doesn't apply to you, point it at anything other than a block device and it'll skip that part.  You still want to provide it a root folder for all the data, since /var/lib/docker and /var/lib/etcd get mapped there.

## k8smaker_precondition
This script is executed on the `control-node` when it is initializing the k8s cluster the first time.  It is also executed on each worker as it is added.  Consequently, it's a great place to throw additional tweaks to the configuration if you want.

## k8smaker_init_aws/baremetal
You run this **on the machine** you want to be the `control-node`.  This script preps the machine, which configures apt, installs a few packages it needs to do the install like curl, docker, kubeadm, kubectl, kubelet, etc.  Several .yaml files are generated by this process and scattered across the file system.  This script does not go out of its way to delete files on disk that might exist, but it will install/upgrade things.  If you already have docker images running on this machine, the /var/lib/docker folder will move, so you are going to want to stop all your containers first, then move the contents of /var/lib/docker to $DATAFOLDER/docker.

One little quirk is that Istio refuses to complete installation without a worker node to schedule some pods onto.  Consequently, the `control-node` always starts off as a full etcd,control,worker role, which means you have a fully working cluster immediately.  There's a setting in the config file to convert it to etcd,control when the first worker node is added.

## k8smaker_addworker_aws/baremetal
You run this on the `control-node`, passing it the **IP or hostname** of the node you want to be a worker for the cluster.  First, it generates a Bash script called at `~/CLUSTERNAME/k8smaker_addworker_HOSTNAME`, so you can inspect what it's doing.  The script then configures the new node with an ssh key the cluster uses for passwordless operation, copies the customized Bash script over, then remote executes it.  You will probably need to accept the SSH signature, mayneed to provide an SSH password, then give the password to allow sudo access on that machine, since some commands require root to configure it.

## k8smaker_drain
Run this on the `control-node`, pass in the **nodename** to drain.  This simply tries to drain pods off the specified node.  It's still part of the cluster, just not in use.  Some things don't drain nicely, but I'll handle that better in the future--hence the script.  If you have a node you want to take down, drain it first.  This will eventually manage re-scaling stateful apps that need to be rescheduled to other nodes, but for now just executes the correct kubectl drain command for you.

## k8smaker_undrain
Run this on the `control-node`, pass in the **nodename** to drain.  This puts a drained node back into service.  Simple as that.  Technically, the right name is uncordon, but it ruins the symmetry of the naming, sorry.

## k8smaker_deleteall
You run this on the `control-node`, passing it the **IP or hostname** of the node you want to completely remove from the cluster.  This constructs a script to run on the specified host, copies and remotely executes it on that host, then attempts to delete the node from the cluster on the off-chance it's unable to reach it (crashed, network down, whatever).  If it successfully executes the script, everything related to Docker, Kubernetes, etcd, and so forth is deleted off the machine and uninstalled.  It's vicious, so be careful.  Always try to drain first, or you may experience data loss depending on your workload.  Note, this will also work on control nodes (you can pass `localhost` to it to wipe the current machine).

## k8smaker_setcontrolnode
You can use this script to change whether a control node will run workloads or not.  It's very simple.  Just pass it the **nodename** of the control node you want to change, followed by either **schedule** or **noschedule**, and it alters the labels and taints to allow kubelet to schedule pods there or not.  Be very careful, though, not to pass in any non-control node names.  It's not safe to do so and may cause problems.

## Disclaimers
Tested on Ubuntu 18.04LTS only.  May run on any reasonably vanilla `systemd` based Debian-based Linux.  Do not run these scripts on anything except a newly installed machine.  **Data loss is very possible.**  Some parts of these scripts run as root.  As with anything you download off the internet, running a script as `root` is dangerous and until it's proven trustworthy, it may destroy anything or everything it touches.  I promise this script doesn't intend to do that, but why should  you trust me?  **Read it over first, and if you see anything you are concerned with, don't run it!**  Even better, fix it and send me a pull request.

## TODO
- Add a script to add an etcd,control node to the cluster, or change init to start with a 1-, 3- or 5-node control plane.
