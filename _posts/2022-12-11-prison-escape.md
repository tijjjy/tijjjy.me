---
layout: post
title:  "KringleCon 2022 Prison Escape"
author: "Tijjy"
---
Hello, this is my write up of the Prison Escape challenge for kringlecon 2022.

The aim of this challenge is to escape a container aka a container breakout. I must admit I did run down a rabbit hole when doing my first attempt but once I realised the answer it became quite obvious.

Below here we have the starting terminal which you see when starting the challenge.
![Alt text](/images/kringlecon2022/prisonescape/prisonescape-01.png "Starting screen")

Typically the first thing I do when I have a shell whether it be a vm or container is check whether I have sudo privileges, most often than not low privilege users are given more sudo privileges than they actually need, this creates a security risk and path for an attacker to escalate to root.

To check what sudo privileges you have can be done with the following command.
{% highlight bash %}
sudo -l
{% endhighlight %}
When you run the command you should see this output.
![Alt text](/images/kringlecon2022/prisonescape/prisonescape-02.png "sudo -l")

Taking a look at the output seen above, We can immidiately see our current user can run any command with sudo requiring no password at all!

Now lets escalate to root with the good ole sudo bash and confirm we are root.
{% highlight bash %}
sudo bash
whoami
{% endhighlight %}
![Alt text](/images/kringlecon2022/prisonescape/prisonescape-03.png "sudo bash")

Upon doing some research into privileged containers I came across an article by trendmicro detailing the --privileged flag for containers and how it can be abused by an attacker to gain access to the host system.  
Article: [why-running-a-privileged-container-in-docker-is-a-bad-idea](https://www.trendmicro.com/en_us/research/19/l/why-running-a-privileged-container-in-docker-is-a-bad-idea.html)

Further down in the article there is a section explaining that a privileged container has the capability to list as mount the host devices.
![Alt text](/images/kringlecon2022/prisonescape/prisonescape-04.png "mounting host devices")

Lets take a look at prison escape container and see if we can list the host devices in /dev.
{% highlight bash %}
ls /dev
{% endhighlight %}
![Alt text](/images/kringlecon2022/prisonescape/prisonescape-05.png "listing host devices")

Listing the devices we can see all the standard devices a linux system would have, what caught my eye was the device "vda", this has the naming standard that I see commonly when creating linux partitions which could be vda, sda, xda from past experience.

Lets go ahead and try to mount the vda device. We can do that with the following commands.
{% highlight bash %}
mkdir /mnt/vda
mount /dev/vda /mnt/vda
cd /mnt/vda
{% endhighlight %}
![Alt text](/images/kringlecon2022/prisonescape/prisonescape-06.png "mounting the vda device")

It looks like it mounted without errors, now lets see whats in it.
{% highlight bash %}
ls
{% endhighlight %}
![Alt text](/images/kringlecon2022/prisonescape/prisonescape-07.png "listed /mnt/vda")

Wow look at that, we have just mounted the host filesystem. Since this container was started with the "--privileged" flag it has the capability to see and mount the host filesystem as we just did.

Lets go ahead and get the flag which resides in the user "jailer"s' private key. Now for the purpose of the challenge I'm not going to show the flag so you can atleast go do the challenge yourself and get the experience of breaking out of a container.
{% highlight bash %}
cat home/jailer/.ssh/jail.key.priv
{% endhighlight %}
![Alt text](/images/kringlecon2022/prisonescape/prisonescape-08.png "catting the jailers private key")

Thats it, we successfully completed the prison escape challenge, now go ahead and do it yourself. I personally liked this challenge as I'm quite a fan of containers!