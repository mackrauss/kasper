---
layout: post
title:  "ODK behind a reverse proxy"
date:   2014-05-09 13:00:00
categories: jekyll update
---

A while ago I started working on a very interesting side project called CHAT which is short for [Community Health worker Assistive Technologies][chat]. Goal of the project is to develop and use tablet based software that supports health education in Sub-Saharan Africa. One of the technologies we use to collect data is the [OpenData Kit (ODK)][odk].

ODK offers you a set of tools of which we use the Android tool Collect and the server called Aggregate. This solution allows us to build forms in XML and distribute them from the server to the tablets and receive the data back on the server in a nice database.

Aggregate runs on Apache Tomcat and while I am not a particular fan of Tomcat, I do appreciate that it allows the ODK team to easily distribute a complete server stack in one WAR file. More importantly, I ran a few Tomcat based services in the past and are not totally unfamiliar with using it.

Setting up the server is not entirely the point of this post since the setup process is explained very well in the documentation of the ODK team for [Tomcat][odk tomcat] and in more detail how to do this on a [AWS instance] [odk aws]. What I didn't like was the fact that the setup of ODK on Tomcat is explained without a Webserver as a reverse proxy in front of it. Instead there are basically two scenarios described. First, you run Tomcat on a custom port (like 8080), which isn't great since nobody likes to type in port numbers. Second, you could use iptables to direct traffic coming in on port 80 to the custom port. While this gets rid of the need to type the port number, it still has some problems.

One problem is that you can't run a websever that servers other sites since the standard http and possibly https ports (80 and 443) are forwarded to Tomcat (and you don't want all your stuff run on Tomcat). Another problem is that I believe that webservers like Apache and Nginx are more battle hardend to deal with the Internet (load, hacking, malformed request). Furthermore, I use the webserver that comes with the distro and installed Tomcat via a download. This means the webserver will get security updates for critical bugs while Tomcat won't in my case. I do this since I don't like how Debian splits up the directories for Tomcat and because it makes it easier to install the Tomcat version recommended by the ODK team and keep it at that. Finally, I find it easier to run a https site on nginx or Apache than setting this up on Tomcat. As a bonus I can show some error page when Tomcat/ODK decides to crash (which is does too often for my tast).

This is a rather long prelude, but I think the thinking process of why I like to put a webserver as a reverse proxy in front of Tomcat is important to understand. Otherwise people will just tell me to put Tomcat on 80 or 8080 and be done. Not what I want and am willing to do.

So what is the problem then? After creating the WAR file and doing the database setup I am able to start up the ODK Aggregate instance and login on via port 8080 (talking directly to Tomcat). If I go through nginx it doesn't work. Here is my nginx setup

{% highlight bash %}
location / {
        proxy_set_header  X_Forwarded_Proto https;
        proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header  Host $host;
        proxy_set_header  X-Url-Scheme $scheme;
        proxy_redirect    off;
        proxy_max_temp_file_size 0;
        proxy_pass http://127.0.0.1:8080;
        proxy_read_timeout 60;
}
{% endhighlight %}

So after googling for some time, changing various settings (nginx, Tomcat, ODK), and writing the mailing list, it became clear that I needed to adjust the *ODKAggregate-settings.jar*. The jar contains a file called *security.properties* which specifies the ports for http and https as 8080 and 8443. I think that once you connect to the ODK instance through nginx, Tomcat will try to redirect the traffic over to port 8080. This doesn't work on my setup since I block all port with ufw that are not absolutely necessary. So I had to edit the file and change the ports. But they are within a jar. I found a [forum post][stack] asking about this and a bit down is the [correct answer][answer] how to do it. Details can be read on the [Java documentation][java doc] site.

Around line 32 I found these settings:
{% highlight java %}
#security.server.port=80
#security.server.securePort=443
security.server.port=8080
security.server.securePort=8443
{% endhighlight %}

And adjusted the setting to this:
{% highlight java %}
security.server.port=80
security.server.securePort=443
#security.server.port=8080
#security.server.securePort=8443
{% endhighlight %}

Now ODK works behind my nginx proxy. Not sure if I did everything 100% correct since submission of form data via SSL seems to not work correctly. So I wouldn't mind if somebody told me what I did wrong and could improve :)


[chat]: http://chat.lmbutler-ssa.net/
[odk]: http://opendatakit.org/
[odk tomcat]: http://opendatakit.org/use/aggregate/tomcat-install/
[odk aws]: https://code.google.com/p/opendatakit/wiki/AggregateAWSInstall
[stack]: http://stackoverflow.com/questions/1224817/modifying-a-file-inside-a-jar
[answer]: http://stackoverflow.com/a/18303058/1382738
[java doc]: http://docs.oracle.com/javase/tutorial/deployment/jar/update.html

