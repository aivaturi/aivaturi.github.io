---
layout: post
title:  "VMWare Lab Manager and Perl"
date:   2009-12-08 21:46:00
categories: vmware vm labmanager 
---

VMware's Lab Manager is an amazing tool to have for testing organizations (which was originally designed by a company called Akimbi). But this post is not about what it is & how it can be useful. This post is to give you an example on  how you can consume its SOAP API from Perl. We use Lab Manager extensively for all our feature testing & automation. For automation, you can deploy, undeploy and change some of the vm properties using the SOAP API. Lab Manager also has more "[internal API][internal-api]", which is not officially supported but can be used for more control of what kind of automation you wanna do. 

Anyhoo, for my group I ended up creating a simple perl module to deploy & undeploy (and delete) library configurations. For SOAP, I used SOAP::Lite, which is very widely used & supported module. The attached module is a working example. Just remember that each request to the LM server requires that you send auth headers (read the LM API documentation for further info). get_auth_header accomplishes that in my module. 

The module has its own POD to help you out. If you optimize it & add more methods to it, please let me know.

Project page: [VMware-LabManager][lm]

[internal-api]: https://communities.vmware.com/community/vmtn/vcenter/labmanager/content?filterID=contentstatus[published]~objecttype~objecttype[document]
[lm]: https://github.com/aivaturi/VMware-LabManager
