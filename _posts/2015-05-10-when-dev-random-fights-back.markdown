---
layout: post
title:  "When /dev/random fights back"
date:   2015-05-10 16:49:00
categories: 
---
A few months ago I reached out to [StackOverflow in utter desperation][SO-ref2]. And ran into something completely unexpected.

I was building a dead simple [Spring Boot][spring-boot-ref] application. I wrapped it up inside a docker container and booted it locally. Everything went up quick with no errors, and testing checked out.

The horror began when I pulled the container onto my [digital ocean][DO-ref] box. The application would hang every time!

###Reading is for squares

I tried swapping java distribution and versions, nothing.

I deployed on boxes with 4x the memory and more CPU, nothing.

I googled and googled and googled. The only thread I could find was this tidy [StackOverflow post][SO-ref1]. And of course being the panicked consulter with a tight deadline I was, I brushed over it, saw that it involved something about changing all `/dev/random` sources to `/dev/urandom` and said "That'll never work for me! Spring Boot isn't even making it to my source code!


Finally, I decided I'd return to the stack overflow post and see if I could discern something useful. The comment on the answer had the key.

*"As I see the root of the problem now, I installed `haveged` and now I have the `/dev/random` entropy source. Many thanks!" - sz4b0lcs*

Well It's worth a try!

added `apt-get install haveged -y` to the docker file and... nothing.

Darn. What am I doing wrong?

An excerpt from the thread dump during the hang:

{% highlight java %}
at java.io.FileInputStream.readBytes(Native Method)
    at java.io.FileInputStream.read(FileInputStream.java:246)
    at sun.security.provider.SeedGenerator$URLSeedGenerator.getSeedBytes(SeedGenerator.java:539)
    at sun.security.provider.SeedGenerator.generateSeed(SeedGenerator.java:144)
    at sun.security.provider.SecureRandom$SeederHolder.<clinit>(SecureRandom.java:203)
{% endhighlight %}

Let's go find that source code and see what the deal is.

{% highlight java %}
byte getSeedByte() {
    byte b[] = new byte[1];
    int stat;
     try {
        stat = devRandom.read(b, 0, b.length);
     } catch (IOException ioe) { ... }
     ...
}
{% endhighlight %}
 *[the source from grepcode][grepcode]*

Well we're definitely stuck on reading the `/dev/random` source.

###Stumbling to a solution

Last resort: I'll try installing haveged on the host, maybe haveged on a container doesn't work right.
{% highlight bash %}
$ apt-get install haveged -y
{% endhighlight %}

And it works! Why? Who cares lets get this job done!

###4 months later...

Wait why did I have to modify my host to get my container working... that seems wrong.

Well it is. There are other solutions that can keep the container isolated from it's host. My favorite so far is to tell java to use `/dev/urandom/` instead of `/dev/random/`. This replaces the blocking random source with the non-blocking version.

Just add the following line to your program invocation:
{% highlight bash %}
-Djava.security.egd=file:///dev/urandom switch
{% endhighlight %}
 *Thanks to the [security stack exchange][sec-article] for this one*

And there are active discussions about whether using `urandom` is actually a massive security flaw or not. See [this article][random-article] for the gory details.

###Pardon the dust
I will update this post with details on how entropy is handled specifically with docker, and why haveged on the docker container didn't work once I finish finals.

Cheers

[spring-boot-ref]: http://projects.spring.io/spring-boot/
[SO-ref1]: http://stackoverflow.com/questions/25660899/spring-boot-actuator-application-wont-start-on-ubuntu-vps
[random-article]: http://www.2uo.de/myths-about-urandom/
[sec-article]: http://security.stackexchange.com/questions/14386/what-do-i-need-to-configure-to-make-sure-my-software-uses-dev-urandom/14387#14387
[SO-ref2]: http://stackoverflow.com/questions/27612209/spring-boot-application-wont-boot-at-startup-inside-docker
[DO-ref]: https://www.digitalocean.com/
[grepcode]: http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/6-b14/sun/security/provider/SeedGenerator.java#SeedGenerator.URLSeedGenerator.%3Cinit%3E%28%29
