---
layout: post
title: "Securing Web Apps With Authelia"
author: "Tijjy"
---
# Introduction
Welcome to another blog post, today I'm going to show you how to secure web applications with Authelia!

Authelia intregrates into many common reverse proxies to add authentication in front of web applications, for example you have a website that you want to implement multi-factor authentication in front of it, Authelia has you covered. Authelia supports single factor and multi-factor, allowing you to authenticate with a one time passcode or even a yubikey!

The reverse proxy I will be using to setup Authelia with will be Traefik as it is a easy to configure and fast reverse proxy.

For this demonstration and walkthrough I will be using Ubuntu 22.04 on a 1vcpu 1GB Memory VPS hosted on [www.binarylane.com.au](https://www.binarylane.com.au/). I highly recommened BinaryLane as they provide fast and reliable VPS solutions and even dedicated servers!

# Prerequisites
- Linux Server
- Docker
- [https://github.com/tijjjy/Authelia-Traefik-Docker](https://github.com/tijjjy/Authelia-Traefik-Docker)]
- Ports 80,443 TCP Open
- DNS Record for Authelia sub-domain
- DNS Record for web application

# Setting the DNS Record
Before we configure Authelia, we need to create a DNS record point to our server so that we can request a certificate from letsencrypt.

Log into your Domain registrar and create an A record for the domain you want to use such as derp01.example.com thats points to the IP address of your server.  

For example:
- auth.example.com
- A Record
- 1.1.1.1 (Using your own servers IP address of course)

Once set, confirm the domain resolves to the IP address by running,
```bash
dig auth.example.com
```
OR
```bash
nslookup auth.example.com
```
Either command should return the IP address of your server.  

Repeat the previous steps for your web applications domain.  

# Installing Requirements
To get started lets update the system which on Ubuntu can be done with the following commands.
```bash
sudo apt update
sudo apt upgrade -y
```
After the upgrades are installed it's always best to reboot the server to reload into the latest linux kernel.
```bash
sudo reboot
```
Once your server is fully updated and finished rebooting, lets go and install Docker and Git
```bash
sudo apt install docker docker-compose git -y
```
If you have trouble installing docker, you can find instructions for the linux distribution you are using at the official docker docs. [https://docs.docker.com/engine/install](https://docs.docker.com/engine/install/)

# Cloning the Repository
With all the requirements installed, lets go ahead and clone my repository that contains all files and configuration we need to get started.  
```bash
cd /opt
git clone https://github.com/tijjjy/Authelia-Traefik-Docker authelia
cd authelia
ls -lah
```
If done correctly you should see the following

![ls directory](/images/posts/securewebappswithauthelia/picture1.png)

# Configuration

## Traefik Configuration
There are a few sections within the config files that we need to change.

First open docker-compose.yml
```bash
nano docker-compose.yml
```
Change the environment variable TZ in section for Authelia to your own timezone.  

![Authelia Timezone](/images/posts/securewebappswithauthelia/picture2.png)

Now open the traefik config file
```bash
nano data/traefik/traefik.toml
```
At this point we only need to change 3 sections of the traefik config file.

Go to the AUTHELIA ROUTER section in the http.routers section and change line
```
rule = "Host(`auth.example.com`)"
```
to the domain you want to use for Authelia, for this demo I will use,
```
rule = "Host(`auth.tijjjy.me`)"
```

![Authelia Host](/images/posts/securewebappswithauthelia/picture3.png)

Next, go to the #MIDDLEWARES section to the address line and change
```
address = "http://authelia:9091/api/verify?rd=https%3A%2F%2Fauth.example.com%2F"
```
to
```
address = "http://authelia:9091/api/verify?rd=https://auth.tijjjy.me"
```

![Authelia Middleware Address](/images/posts/securewebappswithauthelia/picture4.png)

Lastly, change the email line at the #letsencrypt section to your email that you want to use to register the certificate with letsencrypt.  

![letsencrypt email](/images/posts/securewebappswithauthelia/picture5.png)

Cool, thats it for the the traefik config file for now, let's move on to the Authelia config file.

## Authelia Configuration
A default reference template of the Authelia configuration can be found at the Authelia github which outlines every possible configuration available.  
[https://github.com/authelia/authelia/blob/master/config.template.yml](https://github.com/authelia/authelia/blob/master/config.template.yml)

Before we edit the Authelia config file we need to generate a few random secure strings, run this command 3 times to generate 3 strings.

You can find more information on random alphanumeric strings at the Authelia docs.  
[Generating-a-random-alphanumeric-string](https://www.authelia.com/reference/guides/generating-secure-values/#generating-a-random-alphanumeric-string)

```bash
docker run authelia/authelia:latest authelia crypto rand --length 64 --charset alphanumeric
```
You should see output similar if successful. Take note of the output.

![secure random strings](/images/posts/securewebappswithauthelia/picture6.png)

Now open the Authelia config file,
```bash
nano data/authelia/configuration.yml
```

Replace the variable jwt_secret with the first secure string you generated.  

![jwt_secret](/images/posts/securewebappswithauthelia/picture7.png)

Replace the default_redirection_url variable with the domain you are using for Authelia.  

![default_redirection_url](/images/posts/securewebappswithauthelia/picture8.png)

Next go down to the Session Provider Configuration section and change the secret variable to the second secure string you generated and replace the domain variable to the apex or root domain you are using, for me that is tijjjy.me.  

![session](/images/posts/securewebappswithauthelia/picture9.png)

Next go to the Storage Provider Configuration section and change the encryption_key variable to the 3rd secure random string you generated.  
![encryption_key](/images/posts/securewebappswithauthelia/picture11.png)

That's it for the configuration for now. Well Done!

## Creating an Authelia User
More information on generating passwords for users can be found at the Authelia docs.  
[https://www.authelia.com/reference/guides/passwords/#passwords](https://www.authelia.com/reference/guides/passwords/#passwords)

It's time to create our first user so we can login to our Authelia instance, we will be using the pbkdf2 hashing algorithm to generate our password, lets go ahead and run the command to create our password. Remember to change the 'password' argument to the password you want to use!  
```bash
docker run authelia/authelia:latest authelia crypto hash generate pbkdf2 --password 'password'
```

Now that we have our password hash we want to use, let's open the users_database.yml file and create our user.  
```bash
nano data/authelia/users_database.yml
```
Replace "authelia-demo" with the username you want to use, I will use "tijjjy".  
Replace "displayname" with your desired display name.  
Replace "password" with the password hash you just generated.  
And finally, replace "email" with the email you want to use for the your user.  

Your users file should look something like this,  

![default_redirection_url](/images/posts/securewebappswithauthelia/picture10.png)

We now have our user all setup and ready to go!
# First Time Start-Up
All of our initial configuration should be complete, lets go ahead and start out containers.  
```bash
docker-compose up -d
```

Let's go ahead now and open your preferred browser and navigate to https://auth.domain.com replacing auth.domain.com with the domain you set for Authelia.  

![Authelia Homepage](/images/posts/securewebappswithauthelia/picture12.png)

You should see the Authelia homepage, go ahead and login with the user you created, you will be prompted to setup OTP multi-factor authentication. 

One you click on the register button for OTP, open the following file which will contain the initial link to register multi-factor authentication,  
```
nano data/authelia/notification.txt
```

Once you have setup multi-factor authentication, let's go ahead and add our first web application to Traefik and Authelia!

# Adding Our First Web Application
Open the Traefik config file,
```bash
nano data/traefik/traefik.toml
```

Navigate down to the services section and add the following,
```
[http.services.webapp]
  [http.services.webapp.loadBalancer]
    [[http.services.webapp.loadBalancer.servers]]
      url = "http://whoami:2001/"
```
Replacing the each reference to webapp with the name you want for the service
Replacing the "url" with the url or address of the web application you want to protect, since I'm protecting another container in the same network, I can just put the container name and it's port.  

![webapp](/images/posts/securewebappswithauthelia/picture13.png)

Now lets add the http router, go to the http.routes section and add the following,  
```
[http.routers.webapp]
  entryPoints = ["https"]
  rule = "Host(`webapp.tijjjy.me`)"
  middlewares = ["authelia"]
  service = "webapp"
  [http.routers.webapp.tls]
    certResolver = "letsencrypt"
```
Once again replacing "webapp" with the name you want to call the router.  

Make sure for the service line, to add the name of the service that you gave your web application, in my case it was "webapp". 

Replace the line,  
```
  rule = "Host(`webapp.domain.com`)"
```
With the domain name you want for your web applications. Make sure the domain name is set correctly in your DNS registrar!!!

It should look something like this,  

![webapp http router](/images/posts/securewebappswithauthelia/picture14.png)

Now lets open the Authelia configuration and add our web application.  
```
nano data/authelia/configuration.yml
```

Go to the rules section, and replace the default rule with your the domain of the web application you added to Traefik, it should look like this.  

![Authelia webapp](/images/posts/securewebappswithauthelia/picture15.png)

# Accessing Protected Web Application
Epic, we should be all set and ready to restart our containers and login to our newly protected web application. Let's go ahead and restart our containers. Make sure to save your configurations!  
```bash
docker-compose down
docker-compose up -d
```

Open your preferred browser and navigate to the URL for your web application, for me it is https://webapp.tijjjy.me  

As you can see by the URL, we have been redirected back to Authelia and asked to login, let's do that.  

![Authelia webapp accessing](/images/posts/securewebappswithauthelia/picture16.png)

Once logged in you will now be accessing your protected web application!!!

![Authelia webapp accessing](/images/posts/securewebappswithauthelia/picture17.png)

# End

Awesome work, you are now protecting your web application, you can go ahead and add more web applications to Authelia and Traefik to protect even more services. I definitely encourge you to check out the Authelia and Traefik documention on how to do even more advanced configurations.

I hope you enjoyed this walkthrough and learned how easy it is to protected web applications with Authelia.

Thanks for reading!

# Links
- BinaryLane [www.binarylane.com.au](https://www.binarylane.com.au/)
- Github [https://github.com/tijjjy](https://github.com/tijjjy)
- Github Repository [https://github.com/tijjjy/Authelia-Traefik-Docker](https://github.com/tijjjy/Authelia-Traefik-Docker)
- Authelia [https://www.authelia.com](https://www.authelia.com/)
- Traefik [https://doc.traefik.io/traefik](https://doc.traefik.io/traefik/)