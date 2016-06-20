---
layout: post
title:  "Managing Secrets with Vault and Consul (Part II - Secrets management workflow)"
date:   2016-05-13 21:25:00
categories: Encryption Vault Consul
---

In [Part I]({% post_url 2016-05-01-secrets-vault-consul %}), we got an overview of Vault and how it might help us in managing various secrets. To better understand it, let us circle back to our 3-tier architecture again. Assume that the web app needs access to SQS and also the database (MySQL, PostgresSQL, MongoDB etc.).

If your web app is written in Flask, for e.g., you might end up putting AWS access keys in config.py or in environment or under ~/.aws/credentials - in clear. Needless to say, that is exactly what we're trying to prevent. Instead, we will have the Flask app read the secrets from Vault. This means that the secrets (AWS access keys and DB credentials) reside unencrypted only in memory. But to access Vault, the app still needs some sort of auth mechanism. In this post, we present a workflow that is designed to accommodate such a requirement in a relatively safe manner. There are three features of Vault, that we will consider:

- [Generic Secrets Backend] [1]
- [Cubbyhole Secrets Backend] [3]
- [AWS Secrets Backend] [2]

## Secret Mounts
Generic secrets backend is the default secrets backend for Vault. If you already went through the [various secret backends] [3] of Vault, you might notice that there is already a backend for MySQL, ProstgresSQL and Cassandra. And in fact, if you do use any of those, you should use those backends. But for this exercise, let us assume that you will manually create database user and password with appropriate privileges. This is to showcase the most generic solution possible via Vault. Apart from username and passwords, you could use the generic backend to store any arbitrary secret.

If you're unfamiliar with specifics of Vault set up and operations, I urge you to read either the documentation of Vault or go through Part III, which will cover the basic operations with Vault and Consul.

Let us assume you already created a generic secrets mount called "config". The mount point for this backend will always be "config/".  Similarly, let us assume that you also created a mount for AWS secrets backend and Cubbyhole. You may verify the mounts by using Vault CLI:

```bash
$ vault mounts
Path        Type       Default TTL  Max TTL  Description
aws/        aws        system       system
cubbyhole/  cubbyhole  n/a          n/a      per-token secret storage
sys/        system     n/a          n/a      system endpoints used for control, policy and debugging
config/     generic    system       system
```

The mounts will be used in our workflow for managing secrets.

## Workflow to access DB credentials
![Secrets Management Workflow](/assets/images/secrets-mgt-workflow.svg){:class="img-responsive"}

### APP ID and User ID management
The secrets are provided by Vault, and our web app will need a mechanism to access those secrets in Vault. Vault addresses this via [Auth backends] [4]. As these auth activities are machine based, an authentication backend that is more suitable for this scenario is [App ID] [5]. With App-ID auth mechanism, you will need two relatively difficult to guess secrets. For our use case, let us say we pick the following for those auth secrets:
- **For App ID**, we pick randomly generated UUID. It will be generated by some offline config management system like Ansible, Chef etc. This ensures that web app has no real control over APP ID itself and we can frequently rotate it.
- **For User ID**, let us pick the sha1sum of the mac address of eth0, of the instance where the web app is running. This will have to be calculated by the web app as well as our Ansible automation. Let us say if the mac address was "11:ff:30:d6:ba:5f", sha1sum would be: 256e73e33e468e1a679edc9e5647594b789224e5.

### Manage access to APP ID and User ID with Cubbyhole
The generated App ID will be mapped to the User ID of each instance we deploy in production. That means there will be as many mappings for that one App ID as there are instances provisioned. If you want to be even more paranoid, you could generate APP ID per instance. Using both the App ID and the User ID, the web App will be able to authenticate with Vault and then access all the config secrets. This "access" will be controlled by our policy.

Since User ID is calculated by web App at run time, the other piece it needs to know is App ID itself. Writing that information in a file (on the app server) does not increase the security posture by that much. For one, if the instance is compromised, the attacker has all the information he needs to access your secrets. In most cases, this is the weakest link in your secrets management process. To address this problem, we force the web App to fetch App ID itself from Vault. And this is where we employ the Cubbyhole secret backend of Vault. A unique feature of cubbyhole is that every token gets its own secure enclave,  and no other token (including root token) can access it.

If you're already familiar with "num_uses" option of tokens in Vault, it becomes a very convenient and required tool for our workflow. The Ansible automation responsible for generating App ID for a tenant, also generates a "temporary" token with num_uses set to something small, which we will elaborate further. The automation will use that token to write the App ID to the temporary token's cubbyhole and thus, only this temporary token can read that App ID. The automation then copies the temporary token to the app servers and restarts web app.

 All these steps exhaust the total number of uses allowed for the temporary token. To recap, following are the "uses" of the temporary token:
- One write operation to the cubbyhole (automation)
- One read operation to verify the cubbyhole content (automation)
- One auth to cubby hole by the app server
- One read by app server, which get the APP ID.

Once the "uses" are exhausted, Vault automatically revokes the token and no one can read the App ID at that path any more. To summarize, here is what the workflow achieves:

- Keeping secrets a secret
- Relatively secure way to access the store where the secrets are stored
- Frequent (or rather aggressive) rotation of authentication keys
- Supports end-to-end automation of secure access to secrets.

[1]: https://www.vaultproject.io/docs/secrets/generic/index.html
[2]: https://www.vaultproject.io/docs/secrets/aws/index.html
[3]: https://www.vaultproject.io/docs/secrets/cubbyhole/index.html
[4]: https://www.vaultproject.io/docs/auth/index.html
[5]: https://www.vaultproject.io/docs/auth/app-id.html
