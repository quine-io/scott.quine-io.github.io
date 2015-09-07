---
layout: post
title:  "When /dev/random fights back"
date:   2015-05-10 16:49:00
categories: 
---
A few months ago I reached out to [StackOverflow in utter desperation][SO-ref2], and ran into something completely unexpected. This is a post written as a response to [Maleck13][Maleck-ref], a user on Stack Overflow who inquired about my thought and debugging process.

I was building a dead simple [Spring Boot][spring-boot-ref] application. I wrapped it up inside a docker container and booted it locally. Everything went up quick with no errors, and testing checked out. The next step was to deploy the sucker to what would become the production server. That is when the horror began.

I pulled the container onto my [digital ocean][DO-ref] box, did a `docker run` just as I had on my local. This was my view:

{% highlight java %}
...
2014-12-22 23:26:58.957  INFO 1 --- [ost-startStop-1] o.s.b.c.embedded.FilterRegistrationBean  : Mapping filter: 'hiddenHttpMethodFilter' to: ...
{% endhighlight %}

And nothing more. Every time the application would boot, it would get stuck mpping this filter. I found this exceptionally weird as I thought docker was supposed to make my application completely independent of the host it was running on. Fresh OS, no conflicting dependencies, what's the deal? I panicked and started grasping at straws.

Like the good engineer I thought I was, I turned to google. The only thread I could find was this tidy [StackOverflow post][SO-ref1]. And of course being the panicked consulter with a tight deadline I was, I quickly brushed over it, seeing that it involved something about changing all `/dev/random` sources to `/dev/urandom`. I thought *"That'll never work for me! Modifying the source of Spring Boot is clearly not the solution here."* 

I tried swapping java distributions... nope thats not it.

I deployed on boxes with 4x the memory and more CPU... Nothing.

I returned to the stack overflow post to see if I could discern something useful. The comment on the top answer had the key.

*"As I see the root of the problem now, I installed `haveged` and now I have the `/dev/random` entropy source. Many thanks!" - sz4b0lcs*

Well It's worth a try!

added `apt-get install haveged -y` to the docker file and... nothing.

Darn. What am I doing wrong?

I decided it was time to debug the right way.

I knew the program was hung, so if I can find out exactly where it was hung the reason why will probably become obvious.

Fortunately java makes this really easy. Just open up a new bash session and issue the following:

{% highlight java %}
$ kill -3 PID   // PID is the process id of the java process.
{% endhighlight %}

Kill will send a `quit` signal to java. The JVM knows that when this happens, we'll be wanting the stack print in the logs.

If all goes well java should dump out a stack of exactly where it was when the kill came in.

An excerpt from the thread dump during the hang:

{% highlight java %}
at java.io.FileInputStream.readBytes(Native Method)
    at java.io.FileInputStream.read(FileInputStream.java:246)
    at sun.security.provider.SeedGenerator$URLSeedGenerator.getSeedBytes(SeedGenerator.java:539)
    at sun.security.provider.SeedGenerator.generateSeed(SeedGenerator.java:144)
    at sun.security.provider.SecureRandom$SeederHolder.<clinit>(SecureRandom.java:203)
{% endhighlight %}

Let's go find that source code and see what the deal is. Foruntely java makes this easy too, as we get the class and line number. A quick google search of `sun.security.provider.SeedGenerator` had me heading in the right direction.

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

Just like the Stack Overflow article mentioned, we are stuck reading the `/dev/random` source.

`/dev/random` is a blocking source of random data. If the system doesn't have any random data, a call to read this file will block while it waits for the system to generate more random data from its sources of entropy.

The non-blocking alternative to `/dev/random` is `/dev/urandom`. It will generate more psuedorandom data if needed.

###A half baked solution

Stressed out me: 

*"I'll try installing haveged on the host, maybe haveged on a container doesn't work right."*
{% highlight bash %}
$ apt-get install haveged -y
{% endhighlight %}

*"And it works!"*

And it did work. Spring Boot immediately unfroze. My methods had been mapped, and all the tests ran like a charm.

*"Why did that work? Who cares lets get this job done!*"

###4 months later...

Wait why did I have to modify the host OS to get the container working... that seems wrong.

Because it is!

There are other solutions that can keep the container isolated from it's host. My favorite so far is to tell java to use `/dev/urandom/` instead of `/dev/random/`. This replaces the blocking random source with the non-blocking version.

Just add the following line to your program invocation:
{% highlight bash %}
-Djava.security.egd=file:///dev/urandom switch
{% endhighlight %}
 *Thanks to the [security stack exchange][sec-article] for this one*

And there are active discussions about whether using `/dev/urandom` is actually a massive security flaw or not. See [this article][random-article] for the gory details.

Cheers

[spring-boot-ref]: http://projects.spring.io/spring-boot/
[SO-ref1]: http://stackoverflow.com/questions/25660899/spring-boot-actuator-application-wont-start-on-ubuntu-vps
[random-article]: http://www.2uo.de/myths-about-urandom/
[sec-article]: http://security.stackexchange.com/questions/14386/what-do-i-need-to-configure-to-make-sure-my-software-uses-dev-urandom/14387#14387
[SO-ref2]: http://stackoverflow.com/questions/27612209/spring-boot-application-wont-boot-at-startup-inside-docker
[DO-ref]: https://www.digitalocean.com/
[grepcode]: http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/6-b14/sun/security/provider/SeedGenerator.java#SeedGenerator.URLSeedGenerator.%3Cinit%3E%28%29
[Maleck-ref]: http://stackoverflow.com/users/425810/maleck13 
