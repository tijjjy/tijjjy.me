---
layout: post
title:  "Generate a self-signed SSL certificate and key"
author: "Tijjy"
---
To generate a quick and easy self signed certificate and private key, run the following command with any system that has openssl installed.

{% highlight bash %}
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -sha256 -days 365 --nodes
{% endhighlight %}

There is a code snipped of the command here -> [snippet](https://gitlab.tijns.net/-/snippets/2/raw/master/certkeygen.sh)