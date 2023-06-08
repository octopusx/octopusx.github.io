# HA in the homelab
In general I like things simple. I need technology to work for me and provide value without having to dedicate extraordinary amount of time to it. When it comes to running a homelab this is especially true, since I am likely spending my own free time setting things up and maintaining my infrastrcuture. At a certain point in the life of my homelab however I found myself hosting services for my wife, family and some friends. This meant that I started considering a new challenge, the challenge of keeping my services up so that the people that rely on them do not face unpleasant disruptions.

## The Humble Beginning
In the beginning I hosted everything on a lone Unraid server, which I built by converting an old desktop machine I had laying around, a very common beginning for a homelab. Unraid gives you a very economical way to manage mass storage with some redundancy, with the ability to run docker containers and mount their volumes to either the spinning rust array, or an ssd cache drive.

This has worked ok for a few years, but in time 3 problems started grating on me:
- If you want to do any maintenance of the server, you must spin the disk array down. And by maintenance I meana changing most basic settings, such as network or storage configuration. And every time you stop the array, you must shut down the VM and docker engines for the duration.
- There were multiple times where failure to resolve dns meant that my containers would fail to update, then promptly "disappear", together with all their configuration. These were configured manually and now had to be rebuilt manually...
- Every time one of these events occured, all of my homelab users would be inconvenienced. Long periods of unavailability is unacceptable if you want to build a system that multiple people rely on on daily bases.

## Make it double
I decided to experiment with a HA setup and build a second unraid server. This involved abandoning Unraid's built in docker engine (for the most part), and choosing an alternative way to orchestrate my workloads.
My first choice was Microk8s from Cannonical, as it was the simplest way to create a HA multinode cluster.

The goal of this setup was to allow me to take one of the nodes down for "maintenance" and have the services remain available, running on the second node.

The initial idea for how to design this system was basically a mirror setup. Each of the Unraid nodes would have:
- 2 large HDDs for the main Unraid array, storing the data
- 1 NVME SSD used as a separate "cache pool" for storing the Unraid VM and Docker engine data
- A docker container running Pihole
- A number of VMs running a Kubernetes cluster

### What are we running
The most important application that we want to host is Nextcloud, with the intention of using it as photo backup for all family phones. To achieve our functionality we needed to run at minimum the following applications in our kubernetes cluster:
- Metallb
- Traefik
- Nextcloud Server
- Redis
- Mariadb

### Storage
3 out of the 5 services needed are actually stateful applications with requirements to have some form of either object or block storage. The most obvious solution that comes to mind is to leverage the NFS server hosted on our 2 Unraid servers. It was not a viable option because it would again create a single point of failure. You can not run pure NFS with any sort of failover or HA, which means that it doesn't matter that you have 2 unraid servers, the moment you take the wrong one down for "maintenance" the whole system collapses.

I started researching different different distributed filesystem solutions that can be used to provision storage for kubernetes nodes, and I came across the following:
- CephFS + Rook
- GlusterFS
- Longhorn

I have looked into Ceph for a little while before deciding it is too complex for my simple setup.
I tried to setup GlusterFS but failed to make it work. I also found information that the GlusterFS Kubernetes provisioner was getting deprecated, which in the end led me to just one realistic choice, Longhorn.
I looked at Longhorn last because I was hasitant to use a non-generic storage solution from Rancher. I generally try to stick to CNCF and proven open-source projects in my homelab, as I did not want to invest time into setting up infrastructure that may get "deprecated" or "abandoned" and have to re-build everything.
However, I did manage to make longhorn work on my MicroK8S cluster, and I was off to the races with my new HA setup.

The Longhorn documentation mentions that for best results they recommend a high IOPS NVME drive for storage, so I added a couple of NVME SSDs into each of my Unraid servers to pass through to the VMs for that purpose.

### First attempt
For simplicity's sake, I decided I not to create an overly complex and maintenance-heavy system and stayed away from fully featured enterprise-grade K8S solutions. Having played with MicroK8S from Cannonical for a while in a local VM I decided to give it a test run in cluster mode.
I have initially created a very simple 2 virtual machine setup, one machine on each Unraid server.
I passed each its own NVME disk and exposed it to Longhorn and everything was fine... For a while... Until it wasn't fine anymore.

The first time I went to reboot one of the Unraid servers the MicroK8S cluster desintegrated.
In order to maintain operation of the system I had set Longhorn to have 2 replicas of each persistent volume, so one on each node. This means that when one of the nodes comes back after being unavailable for a while, it will discard its copy of the volume and sync it from the nodes that remained available in the cluster. This syncing operation is very IOPS and CPU heavy, to the point that while the sync operation was taking place, the K8S control plane (ETCD) was starved for cycles and the cluster had gone out of sync entirely and failed to come back.

Some of you might say, well, you should have known this right? This is why you shouldn't host the control plane and data plane on the same host! It is asking for trouble... Ok, what should we do about it then? To save some words, I tried creating separate MicroK8S nodes for the control plane, data plane and storage, so that I woud have one of each type of nodes, each on a separate VM across the 2 Unraid boxes, and TLDR, it doesn't work because MicroK8S doesn't let you choose which nodes are supposed to act as masters and which are to be workers. The first 3 nodes to be added to the cluster would form the control plane, as necessary to reach quorum in ETCD. Each next added node would not be a part of the control plane. The moment you reboot one of the "master" nodes though (as long as another node in the cluster is available) another one will pick up that role and you have no say in which one that would be. After a few weeks the cluster has died in much the same way as it did initially. We needed a different solution...

### Move to K3S
It seems to be a bit of a theme at this point, but Rancher Labs seems to know what they're doing. Aside from MicroK8S, their K3S distribution of Kubernetes is another viable and likely the most popular option for Kubernetes on the edge and in the homelab. It is lightweight and flexible, allowing you to dedicate nodes specifically to the control plane and even using alternative data stores for control plane state.

### Final touches
Just like before, we have built a 6 VM Kubernetes cluster, 2 master nodes, 2 workers and 2 longhorn storage nodes. To keep the system as vanilla as possible I elected for the default storage technology, which is ETCD. Such a system however is not HA, to have quorum you need 3 ETCD hosts. To ensure that the cluster stayed up while one of the Unraid servers is taken down I decided to host the third ETCD server... on a spare RaspberryPi I had laying around of course!
