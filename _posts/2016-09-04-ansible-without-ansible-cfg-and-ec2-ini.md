---
layout: post
title:  "Ansible without ansible.cfg and ec2.ini"
date:   2016-09-04 21:25:00
categories: Ansible AWS
---

So you have all your infrastructure in AWS and you chose ansible to manage the said infrastructure. You played with Ansible and started to do more advanced stuff with it. And then slowly come to the realization that you have to hold on to two crutches : [ansible.cfg] [1] and [ec2.ini] [2]. But ideally, you want that to be dynamic as well - may be set all the configs via environment variables.

This is mostly possible without much modification. In [constants.py][3] of Ansible source, you can see all the environment variables, it already recognizes. This allows you to mostly get rid of ansible.cfg. Now, ec2.ini doesn't afford us any such luxury. It is used by only one script : [ec2.py][4]

## Making ec2.ini dynamic

The goal is very simple - make ""ec2.ini" dynamic. And that can be achieved by converting all the config vars in ec2.ini to ENV variables and then read them in ec2.py. It is a standalone (and rather straightforward) python script. Hacking that should be easy. But, if you don't want to touch it explicitly, you can of course subclass Ec2Inventory() in ec2.py and override read_settings() method. An example of how you would modify that method:

{% gist 0de2bd93bf054da6d2b7525a3aafcd45 %}

The above code snippet is rather simple. The only caveat being, I put sane defaults for things I don't use. For e.g. there is no need for elasticache in my personal test environment. You should go through each individual ENV var setting and modify it, if you use those services.

## Setting up the environment

Now that your dynamic inventory configuration is also dynamic, time to automate the setting up of the environment as well. I use a simple bash script for that:

{% gist 01ffcd9e8e0721ad65e95e8a79e09e1a %}

Simply source the script in a bash shell.

In the above script, we do some extra steps to make our lives simpler. Since your ansible host is deployed in AWS, most of the instances will come with a mechanism to detect what region that instance is in. In Ubuntu, you can find /usr/bin/ec2metadata. This helps us set the EC2_REGION env var, which will be used in our read_settings() method snippet above.

Apart from that there is one extra step I do. Setting up "all" group_vars file. This typically contains variables that apply to your entire environment. The last 4 lines render that file in group_vars folder. Here is an example of all.tmpl:

```bash
ssh_remote_port:  ${ANSIBLE_REMOTE_PORT}
ansible_ssh_private_key_file: "${ANSIBLE_PRIVATE_KEY_FILE}"
ansible_dir: "${ANSIBLE_DIR}"
```

You can of course add other global vars in that template.

We have done the following so far:

- Got rid of ansible.cfg
- Got rid of ec2.ini
- Wrote a simple "setenv" script to set up the ansible environment in a single step.

## Consul and Vault

At this point, you might wonder about two things I glossed over:

1. What about the values for ENV vars themselves? Don't we have to hardcode them? How is it going to be different from ansible.cfg or ec2.ini?
2. What about AWS API keys?

For (1), there is indeed a decent solution : [Consul][5]. The "setenv" script should be modified to read all those values from consul. Here is an example:

{% gist 73be347d6ab0d9db5a3873adfb219103 %}

The above snippet makes our "setenv" truly dynamic. All settings can be controlled from Consul and if you need to change any values, change it there and run your ansible command. As long as your playbooks are designed for it, they will simply work.

For the second question - the solution is [Hashicorp Vault][6] (not to be confused with Ansible Vault), specifically: [AWS Secret Backend][7]. Make sure that your lease_duration is small enough so that the keys are automatically revoked by Vault sooner than later. This also has a really good side effect of you being conscious about security.

[1]: https://github.com/ansible/ansible/blob/devel/examples/ansible.cfg
[2]: https://raw.githubusercontent.com/ansible/ansible/devel/contrib/inventory/ec2.ini
[3]: https://github.com/ansible/ansible/blob/devel/lib/ansible/constants.py#L153
[4]: https://github.com/ansible/ansible/blob/devel/contrib/inventory/ec2.py#L216
[5]: https://consul.io
[6]: https://vaultproject.io
[7]: https://www.vaultproject.io/docs/secrets/aws/index.html
