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
sudo apt update && apt install -y wireguard resolvconf ufw
```

It is probably also a good idea to enable firewall on this host. For now, we can safely block all traffic except ssh and wireguard ports:

```bash
ufw allow ssh
ufw allow 51820/udp
ufw enable
```

If you've run the above and are still able to connect to your server over ssh, you've done well. If your server is now turning a blind eye to your ssh connection attempts, you may have to access it over linode shell, then double check you've blocked the correct ports and fix the firewall rules if necessary.

### Configure Wireguard
Once we got the software cosily installed we have to configure the bad boy. We can start with generating a keypair for the server.
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
DNS        = 192.168.1.10,my-search-domain.com
SaveConfig = false
ListenPort = 51820
PrivateKey = mysupersecretgeneratedprivatekey

[Peer]
PublicKey = mylesssecretbutalsosecretpeerpublickey
AllowedIPs = 10.0.0.2/32, 192.168.1.0/24 # The networks that traffic can be routed to over this peer connection
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
sudo apt update && apt install -y wireguard resolvconf
```

Just like before, we will need some freshly baked key material, straight from the oven:
To do that run the following command on your cloud instance:
```bash
wg genkey | tee private.key | wg pubkey > public.key
```

I called my VM and configured it to have the hostname of `wireguard-gateway`, so we will refer to it as such from now on.
Yup, nothing more to see here, let's move on...

### Configure a permanent tunnel
On the `wireguard-gateway` VM we will need to setup the basic wireguard tunnel peer config with a few small extras.
Let's take a look at what my config includes:

```bash
[Interface]
PrivateKey = privatekeyofthewireguard-gateway
Address = 10.0.0.3/24
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE; ip6tables -A FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE; ip6tables -D FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = publickeyofthewireguardlinodehost
Endpoint = my-linode-wireguard.com:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 20
```

Well now that finally looks like some black magic. If you're familiar with `iptables` it likely makes perfect sense, I had to learn through strategic use of duckduckgo and trial-and-error...

The important bits here are the `PostUP` and `PostDown` commands, which are hooks to be executed when the tunnel is set to be up or down respectively. Here we are:
- Enabling forwarding of any traffic that enters through the wireguard interface `wg0`
- Enabling NAT Masquarading when traffic is forwarded to `eth0` to allow it to find its way back to us
Then the `PostDown` reverts the changes, to make sure routing works correctly also when the tunnel is down.

Since this machine is going to be the one establishing the connection with the Linode wireguard host as it itself lives behind NAT, we want to set the `PersistentKeepalive = 20` here too. Wireguard's noise protocol will not transfer any data if whatsoever unless actual data is requested or sent by peers on either side of the connection, so your host behind the layer of NAT will become unavailable after a little while unless you artificially keep pinging it via this setting, extending the lifespan of this connection.

- Enable forwarding
- Use systemd to start and enable the wg service, make sure it auto-reconnects on reboot

## The rest

<!-- - Enable forwarding
- Use systemd to start and enable the wg service, make sure it auto-reconnects on reboot -->

### Client config on Ubuntu

### Client config on Android

### Client config on iOS

## Epilogue