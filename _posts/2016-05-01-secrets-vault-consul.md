---
layout: post
title:  "Managing Secrets with Vault and Consul (Part I)"
date:   2016-05-01 19:25:00
categories: encryption vault consul
---

Every software as a service needs to manage secrets. These secrets can be something like database credentials or AWS access keys or Third party API tokens, or anything you deem important and need safe guards. Simply put, any information that you don't want others to access,  would be a secret. And somehow, you have to safely manage it.

There are various ways to address management of secrets, and you could always **get by** using [symmetric cryptography] [2] as a library. If you're a Python fan, you could check out [cryptography library] [3]. The reason I emphasized "get by" is because, if you deal with any sensitive data, and if you fall under any of the compliance regimes (PCI, PHI, SOC2 etc.), that straight forward simplicity will most probably be inadequate. For one, you won't have a comprehensive story for managing the encryption keys. You might eventually end up with something more serious like: [HSM] [1], but until you get there, you need a better story.

All secrets management techniques boil down to encryption and management of the keys that are used to encrypt and decrypt the data. And even if you do use HSM, you still have to find a solution to  ensure that someone doesn't access the HSM without proper authentication and authorization. And if you don't use HSM, how do you make sure your encryption keys are secure? There has to be middle ground, which ensures that your strategy to handle secrets could be considered rigorous in most cases, doesn't bog you down with key management headaches, and also allows you to expand on that initial solution.

## What do you want to encrypt?

Before we jump in to discussing any encryption solution, you have to know what you're trying to encrypt. Data could be anything like - AWS access keys or something in the database like credit card numbers. Your preliminary task is to determine:

- What should be encrypted?
- What could be encrypted?
- What should not be encrypted at all?

Encryption adds overhead - both in terms of compute and key management. So you should be very clear on what you need to encrypt. For e.g. in a database where you have two tables, one with entries for an individual (Name, Address etc) and another table with Credit Card information. Now, it is a no brainer that all the entries in the Credit Card table **should** be encrypted. But, in the individual table, you **could** encrypt username or email address. And is there any value in encrypting an individual's name? I don't think so, but then again, your policy might dictate that it should be.

Making these decisions upfront will not only affect the implementation of various (micro) services in your infrastructure, but also determine your entire security policy and how it affects key management.

## Encryption Service
Data encryption, encrypted data storage and encryption key management is not something the application should be concerned with.  And more over,  that is a headache a normal developer should not be messing with anyway. You're better off offloading that headache to specialized services. This is where solution like Vault come in to picture.

At this point, we should visit what secrets we're trying to manage and what is an acceptable management policy? Let us consider a typical environment for a Web App - AWS hosting a typical 3-tier architecture application, and this application consuming some of the AWS services. In the process, the application also wants to keep certain secrets, a secret, for e.g. Data of Birth of the users. In simple terms, our encryption as a service should be able to:

1. Manage AWS access keys, ideally with rotation
2. Store application secrets

If you claim to be a "cloud guy", you should be aware of a company called [Hashicorp] [4]. They develop lot of tools that solve lot of the problems of cloud management in a rather elegant way. But for this discussion, we will consider two of their tools : [Vault] [5] and [Consul] [6]. Yes, there are other solutions, and few are listed on the Vault's site as well. And you could tailor those solutions to achieve a similar solution.

So how does Vault address our requirements?

### API
To keep secrets a secret, you need encryption. As with any other solution in your environment, encryption is better off being a micro service. The only real contract between the applications and the encryption micro service is the API that the micro service exposes.  Vault exposes a [HTTP API] [7]. This API also allows you to manage the various [Secret] [8],  and [Auth] [9] backends. There is are quite a few terminologies you have to catch up with, but a very nice [intro to Vault] [10] is always a good place to start.

### AWS Keys Management
When your solution is either partially or fully in AWS, you will have to deal with AWS API. To access that API, you need AWS access keys. For most day-to-day devops operations, you will need elevated privileges; although pre-defined policies and roles (in IAM) should be part of your solution. But if you don't take care of those access keys, by periodically rotating them, [bad things] [11] can happen.

Vault provides you a rather elegant solution with their [AWS Secret Backend] [12]. The idea is simple - every time you want to use AWS API, you request a temporary AWS access keys from Vault that auto expire, unless you renew those access keys lease. And yes, you can attach any IAM policy to those generated keys as well - ideally they should be very restrictive. For e.g., if your application needs to access a bucket in S3, you'd create a policy that allows access to only that named bucket. Use that policy via Vault to generate temporary AWS access keys. When the lease for those access keys expire, Vault will automatically delete from AWS IAM. Lease for these keys are managed by Vault.

This ensures a very important security practice in production systems - no access **right now**, unless you're explicitly granted a time bound exception. This practice should be applied to both humans and machines.

### Encryption in Transit
The other requirement of our is to store encrypted data in our database. Vault is designed to handle that use case via [Transit Secret Backend] [13].

The Transit Secret Backend allows you to encrypt any data and store that data in our primary data store, which happens to be our database of our 3-tier app. Our main application doesn't have to concern itself with how the data is encrypted, or how are those encryption keys are managed. All it cares about is that it is using a service to encrypt & decrypt data, safely.

### Authentication to Encryption Service
Until this point, we have been discussing various ways you can manage secrets with Vault. But how do you make sure that the applications are authorized to access those secrets? Here is where a rather comprehensive [Auth Backends] [9] and [Access Control Policies] [14] come in to picture. If you've ever dealt with Firewall or Active Directory or AWS IAM or myriad of other policy based solutions out there, Access Control Policies shouldn't be very alien. Though it is not very capable as any of the other examples, it gets the job done. Essentially, the job of the policy is to dictate which secrets you have access to. But, if you wanted a more complex one, you will need some time getting used to HCL and also understand how different policies with conflicting rules will behave.

In majority of the cases, you could use the [Token] [15] auth backend. In case of  applications, a better solution might be to use [App ID] [16] auth backend. This auth backend requires your application to present two relatively hard to guess tokens to Vault to gain access to your secrets. We will consider this backend as part of our solution.

### Secrets and Storage
Now that we've broached actual encryption, authentication and dynamic secrets management, the next thing to consider is what other types of secrets can be managed? Vault supports multiple secret types via [Secret] [8] backends. You're already familiar with one such secret backend - AWS. But, for the rest of the data that you want Vault to manage, you probably want to use [Generic] [17] secret backend, which can store any arbitrary data. This could be something like your application secrets, for e.g. Mandrill keys or database username/password etc. We will look at this in more detail when we consider over overall plan to manage secrets.

You should read the documentation for other types of Secret backends that Vault supports. Now that we looked at the types of secrets, where are these stored? Vault supports three main types of storage backends : Consul (consul), Memory (inmem) and Disk (file). Of these Consul is the only recommended HA solution. Vault scales using Consul. Vault by itself can be set up as HA with Master/Slave configuration. There are other community supported backends, which you can checkout at https://www.vaultproject.io/docs/config/.

## Summary
To summarize, we addressed few questions:

- What do we want to keep a secret?
- How do we want to do it?
- How can we manage access to it?

But, all of this discussion is very theoretical and of course not to mention - lot of links. This is just to set the stage for the actual plan to manage secrets safely via Vault and Consul. In Part II, we will dive in to it in detail so that you have a more or less ready made generic solution that you can use in all your apps.

Before you jump on to the actual solution, you might also want to check out [cubbyhole] [18] secret backend. It will play a crucial role in our solution.

  [1]: https://en.wikipedia.org/wiki/Hardware_security_module
  [2]: https://en.wikipedia.org/wiki/Symmetric-key_algorithm
  [3]: https://cryptography.io
  [4]: https://www.hashicorp.com/
  [5]: https://www.vaultproject.io/
  [6]: https://www.consul.io/
  [7]: https://www.vaultproject.io/docs/http/index.html
  [8]: https://www.vaultproject.io/docs/secrets/index.html
  [9]: https://www.vaultproject.io/docs/auth/index.html
  [10]: https://www.vaultproject.io/intro/index.html
  [11]: http://wptavern.com/ryan-hellyers-aws-nightmare-leaked-access-keys-result-in-a-6000-bill-overnight
  [12]: https://www.vaultproject.io/docs/secrets/aws/index.html
  [13]: https://www.vaultproject.io/docs/secrets/transit/index.html
  [14]: https://www.vaultproject.io/docs/concepts/policies.html
  [15]: https://www.vaultproject.io/docs/auth/token.html
  [16]: https://www.vaultproject.io/docs/auth/token.html
  [17]: https://www.vaultproject.io/docs/secrets/generic/index.html
  [18]: https://www.vaultproject.io/docs/secrets/cubbyhole/index.html
