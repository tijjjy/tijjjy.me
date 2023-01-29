---
layout: post
title: "Self Host Tailscale Derp Server"
author: "Tijjy"
---
# Introduction
Welcome to my post, today I'm going to show you how to deploy your own Tailscale DERP server using a docker container I built myself!  

For this demonstration and walkthrough I will be using Ubuntu 22.04 on a 1vcpu 1GB Memory VPS hosted on [www.binarylane.com.au](https://www.binarylane.com.au/). I highly recommened BinaryLane as they provide fast and reliable VPS solutions and even dedicated servers!

# Prerequisites
- Linux Server
- Docker ([Docker Installation Docs](https://docs.docker.com/engine/install/))
- Git
- [https://github.com/tijjjy/Tailscale-DERP-Docker](https://github.com/tijjjy/Tailscale-DERP-Docker)
- Ports 80,443 TCP Open
- Port 3478 UDP Open
- DNS Record for DERP domain name
- iptables, iptables-persistent
- Tailscale Account ([Create a Tailscale account](https://login.tailscale.com/start))

# Why self host your own DERP server?
There are a variety of reasons why you might want to run your own Tailscale DERP server, including but not limited to:
- Organisations using Tailscale may have requirements regarding where network traffic goes.
- Learning new skills and how DERP works.
- Privacy reasons, you may want to have full control of the infrastructure.

# Installing Requirements
To get started lets update the system which on Ubuntu can be done with the following commands.
```bash
sudo apt update
sudo apt upgrade -y
```
After the upgrades are installed it's always best to reboot the server to reload into the latest linux kernal.
```bash
sudo reboot
```
Once your server is fully updated and finished rebooting, lets go and install Docker and Git
```bash
sudo apt install docker docker-compose git -y
```
If you have trouble installing docker, you can find instructions for the linux distribution you are using at the official docker docs. [https://docs.docker.com/engine/install](https://docs.docker.com/engine/install/)

# Setting Up The Firewall
Now to install the firewall, personally I use iptables but this can be done with any firewall.

Run the command below if iptables is not installed.  
```
sudo apt install iptables iptables-persistent -y
```

Open up ipv4ruleset.rules.  
```bash
nano /opt/ipv4ruleset.rules
```

Paste in the following ruleset.  

**IMPORTANT:** Change REPLACEWITHYOURIPHERE to your home IP address or the IP address that you want to allow SSH access. Copy and paste the line to allow additional IP addresses access to SSH.  
```
*filter

-A INPUT -i lo -j ACCEPT
-A INPUT ! -i lo -s 127.0.0.0/8 -j REJECT

#SSH ACCESS
-A INPUT -p tcp -s REPLACEWITHYOURIPHERE/32 --dport 22 -m state --state NEW -j ACCEPT

#DERP SERVER PORTS
-A INPUT -p tcp --dport 80 -m state --state NEW -j ACCEPT
-A INPUT -p tcp --dport 443 -m state --state NEW -j ACCEPT
-A INPUT -p udp --dport 3478 -m state --state NEW -j ACCEPT

#Allow established connections and DROP everything else
-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
-A INPUT -j DROP

#Create docker chains incase they are not already there
-N DOCKER-USER
-N DOCKER
-N DOCKER-ISOLATION-STAGE-1
-N DOCKER-ISOLATION-STAGE-2

COMMIT

```

Save your changes to ipv4ruleset.rules and run the next 2 commands to apply the ruleset and save to the ruleset used on system startup.  
```bash
iptables-restore < /opt/ipv4ruleset.rules
netfilter-persistent save
```

If your server has IPV6 connectivity, follow the next steps otherwise feel free to skip to where we restart the docker daemon.  

Open up ipv6ruleset.rules.  
```bash
nano /opt/ipv6ruleset.rules
```

Paste in the following ruleset.   
```
*filter

-A INPUT -i lo -j ACCEPT
-A INPUT -s fe80::/64 -j ACCEPT

#DERP SERVER PORTS
-A INPUT -p tcp --dport 80 -m state --state NEW -j ACCEPT
-A INPUT -p tcp --dport 443 -m state --state NEW -j ACCEPT
-A INPUT -p udp --dport 3478 -m state --state NEW -j ACCEPT

#Allow established connections and DROP everything else
-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
-A INPUT -j DROP

#Create docker chains incase they are not already there
-N DOCKER-USER
-N DOCKER
-N DOCKER-ISOLATION-STAGE-1
-N DOCKER-ISOLATION-STAGE-2

COMMIT

```

Again lets save your changes to ipv6ruleset.rules and run the next 2 commands to apply the ruleset and save to the ruleset used on system startup.  
```bash
ip6tables-restore < /opt/ipv6ruleset.rules
netfilter-persistent save
```

To check all rules have been applied, you can run these two commands to check the IPV4 and IPV6 rules.  
```bash
iptables -nL
ip6tables -nL
```

Now we have secured your server allowing SSH access to only your specified IP addresses and allowed new connections to the required DERP server ports. As a backup, it is good to make sure you can connect via KVM in your hosting provider incase your IP changes or you get locked out.  

Additionally, we need to restart the docker daemon to allow it to re-create any internal iptables rules that docker requires to operate.   
```bash
sudo systemctl restart docker
```
# Setting the DNS Record
Before we configure and start our DERP server, we need to create a DNS record point to our server so that we can request a certificate from letsencrypt, the domain is also used in the Tailscale ACL for clients as well.  

Log into your Domain registrar and create an A record for the domain you want to use such as derp01.example.com thats points to the IP address of your server.  

For example,
- derp01.example.com
- A Record
- 1.1.1.1 (Using your own servers IP address of course)

Additionally the DNS record step adding in a record for the servers IPV6 Address.  

Once set, confirm the domain resolves to the IP address by running,
```bash
dig derp01.example.com
```
OR
```bash
nslookup derp01.example.com
```
Either command should return the IP address of your server.  

# Generating a Tailscale Auth Token
To authenticate our DERP container to our tailnet we need to supply an initial authentication token to connect the container to our tailnet, this can be done by going to [https://login.tailscale.com/admin/settings/keys](https://login.tailscale.com/admin/settings/keys).  

Once you are logged into your Tailscale account you will see the "Auth Keys" section with the "Generate auth key..." button as seen below.  

![Tailscale Auth Key Panel](/images/posts/tailscalederpserver/picture4.png)

Go ahead and click on the "Generate auth key...", it will open up a prompt for additional key options, just leave all defaults and click on the blue button at the bottom "Generate Key". 

Copy and note the outputted key somewhere safe, we will use it in the next few steps.  

# Cloning Repo and Setting up the DERP config
Great, now we have an updated server with all the requirements installed. Lets go ahead and clone the repository containing the files for the DERP container.
```bash
cd /opt
git clone https://github.com/tijjjy/Tailscale-DERP-Docker
cd Tailscale-DERP-Docker
ls -lah
```
We now have all the required files to configure, build and run our DERP server container.

All we need to edit is the .env file, as this is where we will set our domain we want to use and our Tailscale Auth Key.
```bash
nano .env
```
OR
```bash
vim .env
```

Now we need to replace two variables,
- TAILSCALE_DERP_HOSTNAME
- TAILSCALE_AUTH_KEY

For the TAILSCALE_DERP_HOSTNAME variable, replace the default value with the domain you set in your DNS records.  

For the TAILSCALE_AUTH_KEY variable, replace the default value with the Tailscale Auth Token you created and noted down before.  

If done correctly your .env file will look similar to the following screenshot.  

![Tailscale Auth Key Panel](/images/posts/tailscalederpserver/picture5.png)

# Building the DERP container
Awesome, we are almost there, now all we need to do it build the docker container, this can be done like so,
```bash
docker build . -t tailscale-derp-docker:1.0
```
# Running the DERP container
Now that we have built our container lets go ahead and run it using docker compose.  
```bash
docker-compose up -d
```

We can monitor the container logs,
```bash
docker logs -f tailscale-derp
```

To test if all is running good and the DERP server has a letsencrypt certificate, open your preferred browser and go to,
```bash
https://derp01.example.com
```
Replacing example.com with the domain you set for the DERP server. If everything is good you will no TLS/SSl errors and see the default page like below,

![Testing DERP https](/images/posts/tailscalederpserver/picture7.png)

# Disabling Key Expiring and Setting Tailscale ACL
To stop the need to constantly keep generatng new auth keys for the DERP server when the current key expires, I would disable the key expiry for the newly created machine in your Tailnet. As seen below, go to your machine, click on the 3 dots and click on "Disable Key Expiry" to disable the key expiration.  

![Disable Key expiry](/images/posts/tailscalederpserver/picture6.png)

Following the Tailscale documention on how to set only use your DERP server, we can go ahead and set the following in our ACL. The snippet below needs to be placed under your other ACL rules, for example, mine is underneath the SSH ACL rules section. If you have trouble setting the ACl, the official Tailscale docs for this can be found here -> [Tailscale DERP ACL DOCS](https://tailscale.com/kb/1118/custom-derp-servers/#optional-removing-tailscales-derp-regions)

Remember to change the "HostName" section in the ACL to the domain you set for your DERP server.  
```
"derpMap": {
	"OmitDefaultRegions": true,
	"Regions": {
		"900": {
			"RegionID":   900,
			"RegionCode": "derp01",
			"Nodes": [
				{
					"Name":     "1",
					"RegionID": 900,
					"HostName": "derp01.example.com",
				},
			],
		},
	},
},
```

# Testing our DERP server
All that is left is to test our DERP server, we can use the built in tailscale ping command to ping a machine on our tailnet and confirm how it's traffic is routed.

```bash
tailscale ping machinename
```

When you run the command, if you see the "via DERP(derp01)" then congratulations, you are now using your own DERP server for relay communications!  

I have blanked the IP and derp server name for privacy reasons.  

![Testing DERP server](/images/posts/tailscalederpserver/picture8.png)

# End
There you go, you now have your own DERP server for Tailscale running in a minimal small docker container. I hope you enjoyed this walkthrough!  

Thanks for reading.  

# Links
- Tailscale [https://tailscale.com](https://tailscale.com/)
- BinaryLane [https://www.binarylane.com.au](https://www.binarylane.com.au/)
- Github [https://github.com/tijjjy](https://github.com/tijjjy)
- Github Repository [https://github.com/tijjjy/Tailscale-DERP-Docker](https://github.com/tijjjy/Tailscale-DERP-Docker)