# Wireguard + Linode + Unraid = Profit
I have been using Unraid as my platform to manage my storage at home as well as hosting a few basic services for me and my family for a good few years now.
Despite having a fairly fast internet connection I was not able to expose those services to the internet however as my home router does not support dynamic DNS.
As an alternative I knew that I could get a small VM with a public IP address which could host a VPN endpoint for me to build a permanent tunnel to my Unraid box.
For a long time I avoided this as setting up your own VPN seemed a daunting and needlessly complicated task. Then I've heard of Wireguard.
It lets you set up temporary and permanent tunnels with ease, it is easy to set up and manage and supposed to provide state-of-the art security.
Join me on a short trip to find out how you can set up your own easy-to-manage VPN and access your home network resources from anywhere.

## The public endpoint
To be able to host a VPN you can connect to from anywhere you will need a machine with a public IP.
As mentioned before, this can be solved by setting up DDNS instead, but if you're like me and you don't have that option this is a solid alternative.
The easiest way to obtain a public IP address is to create a small cheap VM with a cloud provider to host our Wireguard VPN.
In my setup I have used Linode, but you can use anything you like really, so long as it gives you that sweet public IP goodness.

### Provision the host
We are going to build our Linode wireguard VPN on an Ubuntu linux VM, version 20.04.
Our VM should have wireguard available as a package out of the box which make installation super easy:

```bash
sudo apt update && apt install -y wireguard
# TODO: Also enable the firewall on this host
# TODO: Also may need other dependencies, like resolvconf and some others...
```

### Configure Wireguard
Once we got the software cosily installed we have to configure the bad boy. We can start with generating a keypar for the server.
To do that run the following command on your cloud instance:
```bash
wg genkey | tee private.key | wg pubkey > public.key
```
This will create the two key files for you, private and public. The contents of the private one will live in the config on the server while the public one has to be shared with every peer that wishes to connect to this server. Either way, it is a good idea to think about where you're gonna store these keys for safekeeping. I recommend using a secure not in your password manager app. You've got one of these already, right? Right!?

The next step is to create the server configuration for your public endpoint.
The location of all of the endpoints will be stored in `/etc/wireguard`. We are going to call ours `wg0.conf`. You will notice that once we are done there will be a new virtual network interface on your linux host also called `wg0`. That's because wireguard will create a virtual interface per tunnel endpoint that you configure, but more on that later ;)

Take a look at the following example config file:
```ini
[Interface]
Address    = 10.0.0.1/24
DNS        = 192.168.1.10
SaveConfig = false
ListenPort = 51820
PrivateKey = mysupersecretgeneratedprivatekey

[Peer]
PublicKey = mylesssecretbutalsosecretpeerpublickey
AllowedIPs = 10.0.0.2/32
```

The above block is a good starting point, and we can go over its contents briefly.

The **interface** block is the configuration parameters for our `wg0` virtual network interface. It also configures some server-side parameters, such as what is the address of the DNS server to be used to resolve domain names on our VPN network. We can also observe the port, which is the network port to be used by peers to establish a connection with the server. We will also need to whitelist this address on our firewall and any security group analogs your cloud provider offers. Lastly we have the private key where, yes, you guessed it, we paste the private key that you so diligently saved in your password manager, haven't you?

The second block we can see is the **peer** block. The first field would be a public key. And no, this is not our server's public key, this is the public key of your peer. A peer in this case is any machine trying to connect to our vpn server. Therefore you will likely have many **peer** blocks in this secrion, as many as there are devices you want to allow to connect to your network. The allopwed IPs field will tell the server what range of IP addresses it will accept traffic from should the handshake with using the public key be successful. This will be the IP address of the `wg0` (or whatever you call name it) interface on the peer. If this is a little confusing, don't worry, we'll be able to connect the dots a little later.

### Ready script
[Check out the ready bash script.](https://github.com/octopusx/wireguard-scripting/blob/main/endpoint_host.sh)

## Unraid
Unraid is a very potent platform for managing your home server infrastructure and it makes it extremely easy to deploy services within your home network.
I myself host a piHole and HomeAssistant containers on mine among others.
It also has the ability to install extensions. When trying to figure out an easy and reliable way to set up a permanent tunnel to my Linode wireguard host I stumbled accross a [nicely written extension](https://forums.unraid.net/topic/84226-wireguard-quickstart/) which at first seemed like it would do the trick. After some attempts to make it work for me however I realised that it won't do the trick, I could not get my Unraid box to forward traffic from the VPN to the rest of my network, which meant my internally-hosted services would not be reachable by other devices on the VPN. If you are in a situation where you are able to use DDNS instead though this is likely the way to go!

In the end I decided to go for a small VM instead. Seeing as setting up wireguard on an Ubuntu machine is so quick and simple this went very smoothly.

### Provision the VM
First, use the Unraid UI to spawn a new Ubuntu Server 20.04 VM. I assigned mine 2 vCPUs and 2GB of RAM, which should be plenty enough for the job.
Once you've got it up and running, install the wireguard package just like on the Linode:

```bash
sudo apt update && apt install -y wireguard
```

### Configure a permanent tunnel

## The rest

### Client config on Ubuntu

### Client config on Android

### Client config on iOS

## Epilogue