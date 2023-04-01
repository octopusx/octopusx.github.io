# HA in the homelab

In general I like things simple. I need technology to work for me and provide value without having to dedicate extraordinary amount of time to it. When it comes to running a homelab this is especially true, since I am likely spending my own free time setting things up and maintaining my infrastrcuture. At a certain point in the life of my homelab however I found myself hosting services for my wife, family and some friends. This meant that I started to worry about a new challenge, which was the ability to keep my services up so that the people using them can do so without unpleasant disruptions.

## The Humble Beginning
In the beginning I hosted everything on a lone Unraid server, which I built by converting an old desktop machine I had laying around, a very common beginning for a homelab. Unraid gives you a very economical way to manage mass storage with some redundancy, with the ability to run docker containers and mount their volumes to either the spinning rust array, or an ssd cache drive.

There were 3 big problems:
- If you want to make any maintenance on the server, you must spin the disk array down. Changing network or storage configuratino included. And every time you stop the array, you must shut down the VM and docker engines for the duration.
- There were multiple times where failure to resolve dns meant that my containers would fail to update, then promptly "disappear", together with all their configuration. These were configured manually and now had to be rebuilt manually...
- Every time one of these events occured, all of my homelab users would be inconvenienced. Long periods of unavailability is unacceptable if you want to build a system that multiple people rely on on daily bases.

## Make it double
I decided to experiment with a HA setup and build a second unraid server. This involved abandoning Unraid's built in docker engine (for the most part), and choosing an alternative way to orchestrate my workloads.
My first choice was Microk8s from Cannonical, as it was the simplest way to create a HA multinode cluster.

> Describe our architecture with 2 nodes

The goal was to make sure that services would stay up if one of the hosts were to go down, or in the worst case scenario that the cluster recovers once the second host comes back up.

### What are we running
The most important deployment to this new cluster was Nextcloud, with the intention of using it as photo backup for all family phones. To achieve our functionality we needed to run at minimum the following applications in our kubernetes cluster:
- Metallb
- Traefik
- Nextcloud Server
- Redis
- Mariadb

### Storage
3 out of these deployments are actually stateful applications with requirements to have some form of either object or block storage. The most obvious solution that comes to mind is to leverage the NFS server hosted on our 2 Unraid servers. It was not a viable option because it would again create a single point of failure. You can not run pure NFS with any sort of failover or HA, which means that as soon as the main NFS server becomes unavailable we loose the benefit of running 2 servers.

I started researching different different distributed filesystem solutions that can be used to provision storage for kubernetes nodes, and I came across the following:
- CephFS + Rook
- GlusterFS
- Longhorn

I have looked into Ceph for a little while before deciding it is too complex for my simple setup.
I tried to setup GlusterFS but failed to make it work. I also found information that the GlusterFS Kubernetes provisioner was getting deprecated, which forced me to try the last one from the list.
I looked at Longhorn last because I was hasitant to use a non-generic storage solution from Rancher. I generally try to stick to CNCF and proven open-source projects in my homelab, as I did not want to invest time into setting up infrastructure that may get "deprecated" or "abandoned" and have to re-build everything.
However, I did manage to make longhorn work on my MicroK8S cluster, and I was off to the races with my new HA setup.

### First attempt

Problems with scaling out mk8s - you don't have to manage the control plane but also you don't contorl it.
With a multi node setup when running longhorn to sync data between nodes, it took the control plane down and wasn't able to recover.

### Move to K3S

More control, we are now able to build our own etcd cluster and master nodes on dedicated VMs.
We are also able to create dedicated longhorn storage nodes, so that sync operations can be done quickly without affecting other workloads.

### Final touches

We are using a raspberry pi as the third etcd node. It has nvme usb storage to make sure we don't wear out the sd card.
DNS servers are run outside of k8s, as docker containers on each unraid server to make sure problems with the cluster do not affect basic home network functionality.