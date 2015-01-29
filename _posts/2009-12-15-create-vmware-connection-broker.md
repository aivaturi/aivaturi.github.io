---
layout: post
title:  "Create your own VMWare Connection Broker"
date:   2009-12-15 11:45:00
categories: vmware vm connection-broker 
---

First of all, "Connection Broker" is a highly glorified marketing term to say that this piece of software will allow you to connect to your Virtual Machine based on some "rules" (access, authorization, availability etc and the more "rules" you add, it becomes more complex) and hence the "broker" part. In this post, I'll be concentrating on VMware since I am more familiar with it, but you can pretty much implement this connection broker for the Sun & Citrix solutions as well. It looks complicated, but it isn't, that is mostly because the excellent SDK that they provide.

I first heard of a connection broker when our sales guy was raving about a company called Leostream. At that time, I thought it was really cool but by the time I got hold of their evaluation VM (yes they give out or at least used to provide their connection broker as a VM), I had already developed a prototype using the VI Perl SDK internally to work with our VPN box. And guess what, they just bundled the Perl SDK & provided some cgi front end to provide a polished product. Now, if that was all they provided then they still wouldn't be in business. They actually provide "policy" & "access" mechanisms to those backend VMs, which add value to the overall product. And their product was the first (I think) to tunnel PCOIP. But regardless, underneath it all, the core engine (at least for VMware) is production ready & they used it as it is. That is why, I keep saying - roll out your own connection broker if you don't care about all the bells & whistles. For e.g. why does a university lab require a commercial connection broker? Get some students to intern for a summer & give them this project. It is not only fun, but there are a lot of internet technologies that they'll come across & learn by doing them.

One of the most amazing thing that VMWare did with ESX (and for that matter most of their product line) was to provide an amazing set of API to pretty much do any task (down to bare metal operation) through SOAP. In fact, they themselves use this API internally to manage all the ESX servers (through Virtual Center). I knew this SDK (and still call this) as VI SDK, but for some reason they keep on changing its name. I think the latest monicker is vSphere, but don't get too hung up on it, soon it may become "vUniverse". Lucky for me, their functionality & direction of the API doesn't change - it just evolves. Back to the actual topic on hand. You should be familiar with the various infrastructure products that VMware sells - ESX, Virtual Center etc and what service they provide. If not some of the discussion here might not make any sense. At the heart of these products is ESX, which hosts the actual VMs themselves. And what VI SDK does is to expose all the operations that you can do on a VM (and the ESX themselves), like deploying a new VM, undeploying, powering on & off etc. 

So let us say, you have set up a farm of ESX servers & you have a VC managing all of them. You can use the VI SDK to talk to the VC server & ask it to manage various operations for you. For e.g. you can tell the VC to deploy a new VM and based on its rules & configuration, VC will deploy the VM on an ESX server & give you details relevant to that deployed VM. In a way, it does the load balancing (load as in number of VMs on a particular ESX) for you. You can of course specify where exactly to deploy the VM but why complicate your life? VC is pretty efficient at what it does. Anyhoo, this farm of virtual infrastructure is pretty much useless unless you could automate these operations, which you can using the VI SDK or purchase a connection broker. Unless you are deploying this for day-to-day production use i.e. your workforce is going all virtual, create your own connection broker & save the money & on the way, earn some job security ;).

To get you started, let me give you some snippets of code. Oh BTW, they release a Perl SDK also, which is really cool and it is really well designed. And since it is designed in Perl, hacking it is just that more fun. So the code snippets, I'm about to show you are in Perl. If you want Java or powershell *shudders* scripts there are tons of examples on their developer site. 

Regardless, you have to read their VIPerl documentation. In fact most of these snippets would make sense & seem familiar if you actually read them first. VIPerl SDK uses certain environment variables - you don't have to use them, but for these examples, it is easy to explain using them. 

Once you have VIPerl SDK installed, just use it...

{% highlight perl %}
use VMware::VIRuntime;

Now set up your environment variables.

$ENV{'VI_SERVER'} = 'vm-vc.acme.com'; # This is your VC server or it can be your ESX server too. 
$ENV{'VI_USERNAME'} = "user";
$ENV{'VI_PASSWORD'} = :password"";
$ENV{'VI_PROTOCOL'} = 'https';
$ENV{'VI_PORTNUMBER'} = '443';
$ENV{'VI_DATACENTER'} = 'MY LAB'; # Look this up in your admin panel of either the VC or ESX server. 

my $datacenter = $ENV{'VI_DATACENTER'};
{% endhighlight %}

These three statements do all the magic of reading, validating options and connecting to the VC server.

{% highlight perl %}
Opts::parse();
Opts::validate();
Util::connect();
{% endhighlight %}

Now let us do some simple stuff like finding your datacenter.

{% highlight perl %}
my $datacenter_view = Vim::find_entity_view(view_type => 'Datacenter',
                                            filter => { name => $datacenter });
{% endhighlight %}

And may be get all hosts under this datacenter

{% highlight perl %}
my $host_views = Vim::find_entity_views(view_type => 'HostSystem',
                                        begin_entity => $datacenter_view);
print "<p>Hosts found:</p>";
foreach (@$host_views) {
   print $_->name,"\n";
}
{% endhighlight %}

Now, find the list of VMs assigned for the VI_USERNAME.

{% highlight perl %}
print "<p>VM's found for $ENV{'VI_USERNAME'}:</p>";
# get all VM's under this datacenter
my $vm_views = Vim::find_entity_views(view_type => 'VirtualMachine',
                                      begin_entity => $datacenter_view);
{% endhighlight %}

Now at this point, you have the list of VMs for a particular user and you might want to connect to it. There are two options for it (that I know of) - 1) Connect using VMWare View Client (there is a linux version too!) or connect directly through the browser using VMWare MKS client. The MKS client will be installed automatically when you try to access the console of a VM in the browser (even though support for all browsers is non-existent). I'll show you how to connect via MKS.

So from our previous command, we found all the VMs for the user, let us create a "View Console" link for it. 

{% highlight perl %}
foreach (@$vm_views) {
   my $mks = VirtualMachineOperations::AcquireMksTicket($_);
   print "$counter: <a href=viewConsole.cgi?vwmks=1&cfgFile=".$mks->cfgFile."&port=".$mks->port.
          "&ticket=".$mks->ticket."&host=".$mks->host.">".$_->name."</a><br/>";
}
{% endhighlight %}

OK, lets start with AcquireMksTicket(). This method provides you a ticket (along with other relevant stuff) to access the console of a particular VM and it is time sensitive that means after a while of inactivity, it expires. The viewConsole.cgi page, which you have to create btw (and is attached to this page), will take the relevant MKS parameters & allow you to connect to the VM console through its browser plugin, which happens to be:

{% highlight html %}
<object id="mks" classid="CLSID:338095E4-1806-4ba3-AB51-38A3179200E9"  codebase="https://$host/ui/plugin/msie/vmware-mks.cab#version=2,1,0,0" width="100%" height="100%"></object>
{% endhighlight %}

Note that the classid might change depending on the version of MKS plugin and you should change it accordingly. Try this example first though. So as you can see, this plugin is shipped with every ESX server. Oh one more thing, if you payed close attention to the codebase attribute of the object tag, you'll notice that we're connecting to the ESX server on which this VM is hosted directly and not through the VC.

Well, there you go - you have connected to your VC, accessed all the VMs for the logged in user & allowed your user to connect to your VM. Now, you might ask how do I deploy these VMs for the user? Can I create groups and allow authorized access to it? Well, if you have read this far & understood what is involved then those things shouldn't be too hard - for one, all the necessary methods are provided through the SDK. Just RTFM.

View Vonsole CGI: [viewConsole.cgi][viewConsole]
[viewConsole]: /assets/files/viewConsole.cgi.zip
