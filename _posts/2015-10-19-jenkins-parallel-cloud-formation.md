---
layout: post
title:  "Parallel build-flow job with AWS CloudFormation"
date:   2015-10-19 20:35:00
categories: jenkins build-flow aws cloudformation
---

[Jenkins Build Flow] [jenkins-build-flow] is a rather useful plugin, and even though I have been using it for a while,
every now & then I come across a scenario where I have one more thing to do along with it to make it even more useful.
Let us consider one such scenario - let us say you have a (parametrized) job and you want to execute it in parallel.
And let us say, you were already in AWS and you want to easily scale out your compute resource so that you can process
those jobs as fast as possible. With few extra steps, you can achieve that in a fairly straightforward way.

##### Plugins to install

Install the following plugins on your Jenkins server:

- [Build Flow Plugin] [jenkins-build-flow]
- [AWS CloudFormation Plugin] [jenkins-aws-cf]

{% highlight groovy %}
{% endhighlight %}

[jenkins-build-flow]: https://wiki.jenkins-ci.org/display/JENKINS/Build+Flow+Plugin
[jenkins-aws-cf]: https://wiki.jenkins-ci.org/display/JENKINS/AWS+Cloudformation+Plugin