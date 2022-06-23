# Wireguard + Linode + Unraid = Profit
I have been using Unraid as a platform to manage my storage at home as well as hosting a few basic services for me and my family for a good few years now.
Despite having a fairly fast internet connection I was not able to expose those services to the internet however as my home router does not support dynamic DNS.
As an alternative I knew that I could get a small VM with a public IP address which could host a VPN endpoint for me to build a permanent tunnel to my Unraid box.
For a long time I avoided this as setting up your own VPN seemed a daunting and needlessly complicated task. Then I've learned of Wireguard.
It lets you set up temporary and permanent tunnels with ease, it is simple to manage and supposed to provide state-of-the art security.
Join me on a short trip to find out how you can set up your own easy-to-manage VPN and access your home network resources from anywhere.

## The public endpoint
To be able to host a VPN you can connect to from anywhere you will need a machine with a public IP.
As mentioned before, this can be solved by setting up DDNS instead, but if you're like me and you don't have that option this is a solid alternative.
The easiest way to obtain a public IP address is to create a small cheap VM with a cloud provider to host your Wireguard VPN.
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
This will create the two key files for you, private and public. The contents of the private key will live in the config on the server while the public one has to be shared with every peer that wishes to connect to this server. Either way, it is a good idea to think about where you're gonna store these keys for safekeeping. I recommend using a secure note in your password manager app or an encrypted backup drive. You've got one of these already, right? Right!?

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

The above block is a good starting point, let's go over its contents briefly.

The **interface** block is the configuration parameters for our `wg0` virtual network interface. It also configures some server-side parameters, such as what is the address of the DNS server to be used to resolve domain names on our VPN network. This is useful if you are hosting a DNS server within your lan to resolve internal domain names. 

Another field we can see is the port, which is the network port to be used by peers to establish a connection with the server. For safety reasons (security by obscurity) it is a nice idea to change it to anything else other than the default. 

We will also need to whitelist this address on our firewall and any security group analogs your cloud provider offers. Lastly we have the private key where, yes, you guessed it, we paste the private key that you so diligently saved in your password manager, haven't you?

The second block we can see is the **peer** block. 

The first field would be a public key. And no, this is not our server's public key, this is the public key of your peer. A peer in this case is any machine trying to connect to our vpn server. Therefore you will likely have many **peer** blocks in this secrion, as many as there are devices you want to allow to connect to your network. 

The allopwed IPs field will tell the server what range of IP addresses (subnets) it will accept traffic from should the handshake with the matching peer be successful. This will be the IP address of the `wg0` (or whatever you call name it) interface BUT ON THE PEER. Plus any other network this other peer will expose to us. 

If this is a little confusing, don't worry, we'll be able to connect the dots a little later.

### Ready script
[Check out the ready bash script.](https://github.com/octopusx/wireguard-scripting/blob/main/endpoint_host.sh)

## Reaching the Unraid server
Unraid is a very potent platform for managing your home server infrastructure and it makes it extremely easy to deploy services within your home network.

I myself host a piHole and HomeAssistant containers on mine among others.
It also has the ability to install extensions. When trying to figure out an easy and reliable way to set up a permanent tunnel to my Linode wireguard host I stumbled accross a [nicely written extension](https://forums.unraid.net/topic/84226-wireguard-quickstart/) which at first seemed like it would do the trick. 

After some attempts to make it work however I realised that it won't do the trick, I could not get my Unraid box to forward traffic from the VPN to the rest of my network, which meant my internally-hosted services would not be reachable by other devices on the VPN. Mind you this was the beginning of my journey and I was only learning about wireguard so quite possibly it's entirely  my fault. If you have managed to make it work, more power to you!

In the end I decided to go for a small VM instead. Seeing as setting up wireguard on an Ubuntu machine is so quick and simple this went very smoothly.

### Provision the VM
First, use the Unraid UI to spawn a new Ubuntu Server 20.04 VM. I assigned mine 2 vCPUs and 2GB of RAM, which should be plenty enough for the job. You can adjust this later anyway if you decide it is under or over-provisionned, so no biggie...

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
- Enabling NAT Masquarading when traffic is forwarded to `eth0` to allow it to find its way back to the VPN subnet
Then the `PostDown` reverts the changes, to make sure routing works correctly also when the tunnel is down.

Since this machine is going to be the one establishing the connection with the Linode wireguard host as it itself lives behind the nasty nasty NAT, we want to set the `PersistentKeepalive = 20` here too. Wireguard's noise protocol will not transfer any data whatsoever unless actual data is requested or sent by peers on either side of the connection. This means that your host behind the layer of NAT will become unavailable after a little while unless you artificially keep pinging it via this setting, extending the lifespan of this connection.

- Enable forwarding
- Use systemd to start and enable the wg service, make sure it auto-reconnects on reboot

### Launch the guard of wires

There are a couple of more steps necessary to turn our `wireguard-gateway` into a defacto gateway router. First of all, generic linux will have packet forwarding disabled by default. We will need to enable it and permanently, so that the setting survives any planned (or indeed unplanned) reboots and shutdown.

Go and add the following line to `/etc/sysctl.conf`:
```
net.ipv4.ip_forward=1
```

To check that it worked run the following command:
```
sysctl net.ipv4.ip_forward
```

The expected return value if it worked is this:
```
net.ipv4.ip_forward = 1
```

We should be able to to turn on our wireguard tunnel at this point. And for that, we have a special magic command:

```
sudo systemctl start wg-quick@wg0
```

Why this magic command? `wg-quick` is a utility that comes with wireguard and it lets you manage wg tunnels easliy as a human being and not a soulless robot. The above will not only start the tunnel but also enable a systemd service for it, meaning that the tunnel will remain up even after reboots. Nice.

### Quick reality check
<!-- Here we should talk about testing pinging from both sides and curling stuff -->

## The rest

### Client config on Ubuntu

Now, this one gave me a lot of trouble for a while but I am very happy that I managed to figure it out all on my own, like a propper big boy with access to stack overflow! Let's take a look at my final configuration file first, as I assume that's mainly what you're here for and postpone story time for a moment.

```
[Interface]
Address = 10.0.0.20/24
DNS = 192.168.1.10,my-search-domain.com
PrivateKey = myclientprivatekeypleasedontshare
MTU = 1280

[Peer]
AllowedIPs = 0.0.0.0/0
Endpoint = my-linode-wireguard.com:51820
PublicKey = thepublickeyofmylinodevpswireguarserverendpoint
```

This configuration block lives on my laptop in the usual place, i.e. `/etc/wireguard/wg0.conf`. 

We will start from the **peer** block for a change.
Allowed IPs here should be set to `0.0.0.0/0`, which means that all of the traffic generated by our host will go through the tunnel. We can use this setting to adjust what goes through the tunnel and if, for example, we limit the subnet to `10.0.0.0/24,192.168.1.0/24`, then the VPN subnet and our homelab subnet will be routed through the VPN tunnel, while all other internet traffic will be ommited and shot out straight onto the interwebz.

Endpoint will be identical for the wireguard-gateway and all of the peers that are not the VPS machine itself. It is the public address of the VPS machine, followed by the VPS' public key to allow this peer to authenticate.

The **interface** configuration, as always, refers to the virtual wg0 interface on my laptop. The address will be the IP my client wants to assume within the VPN subnet.

The DNS entry just as before is telling my computer where to reach my internal domain name server and what my private search domain is.

The private key is to be generated and stored safely, as all of the previously generated keys.

### Story time

Last but not least, the `MTU`. This is an optional parameter and was at the center of my problems with this entire setup.
Generally most people do not need to touch this, though hopfully this will be a cautionary tale for anyone experiencing weird issues with their wireguard tunnels.
My first attempt of making my roadwarrior peer working omitted this parameter, as most guides and vanilla setup documentations do not mention needing it. Indeed my VPN tunnel would connect without it. I could ping all of the hosts on the VPN subnet as well as my private subnet behind NAT. I could even resolve DNS. But when I tried to perform any other type of data transfer over the tunnel however, it would fail.

To troubleshoot this I had tested reachability from and to each host on my network, and all would work besides connections to and from my roadwarrior peer.
This would have indicated that either my `wireguarad-gateway` host is not forwarding traffic correctly to the VPN subnet, or that there is an error in the VPN tunnel configuration on the VPS. At the time however I already had one working android peer, which could access and use all of the services hosted in my private subnet via the wireguard tunnel, so neither of the aformentnioned was a likely conclussion.

I have resorted to take an even closer look at what is happening on the network when I try to establish an ssh connectino between my roadwarrior peer and an ssh server in my private subnet. I set up probes listenning for ssh traffic on all 4 contact points:
- the roadwarrior peer
- the vps server
- the wireguard-gateway
- the target server in the private subnet

What I observed was that connection request would make all of the hops starting from the roadwarrior and ending on the targert server. Return traffic originating from the target server however could be observed on the target server itself, the wireguard-gateway, then the vps server, but the trail would end there. This was an indicator that it is strictly a problem with my ubuntu roadwarriror host.

After some frantic duckduckgoing and many a-stackoverflow page overturned, I found our that my probems are sympthomatic for MTU-related issues. This means that my laptop is not accepting packets exceeding certain size, and therefore dropping most of the traffic returning to it from my wireguard tunnel. Setting a smaller-than-default MTU made it work instantly, and every tunnel and virtual interface lived happy ever after...

<!-- TODO: figure out what MTU actually does. I have an idea but am not entirely sure. Also, paste relevant links here as well as wireshark commands. -->

### Client config on Android

<!-- TODO: research wireguard qr code generators -->

### Client config on iOS

## Epilogue