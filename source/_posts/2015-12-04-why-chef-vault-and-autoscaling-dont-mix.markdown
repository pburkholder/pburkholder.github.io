---
layout: post
title: "Why chef-vault and autoscaling don't mix"
date: 2015-12-04 10:04:06 -0500
comments: true
categories:
---
\[Note: The opinions here are mine and do not reflect a position of my employer, Chef Inc.\]

I received this question through our support team:

> We’ve got a handful of cookbooks that rely on Chef Vault to pick up some secrets – passwords, SSL certs, etc. When we bootstrap new nodes with vault-dependent cookbooks in the run list, the chef-client run fails because the node doesn’t have access to the vault. There seems to be a ‘chicken and egg’ scenario in that we can’t seem add a new node’s host key to a vault during bootstrap – adding an existing node to a vault happens with a knife search, which requires a node to have successfully completed a chef-client run and be registered with the server, but our fresh nodes haven’t done this (and fail due to this…). I think you can see where I’m going with this. I’ve found some discussions around this on the web and have seen that knife bootstrap appears to have some switches for vault, so I tend to believe what we’re trying to do is possible – just haven’t had any success.

> Our new nodes are launched on \[a cloud\] by an automated process ..., but during the customization step of provisioning (cloudinit perhaps?) the bootstrap is kicked off on the node and the run list is converged.

Here's my response:

# ChefVault and knife bootstrap

I'm going to assume you are already familiar with the [theory behind chef-vault](https://github.com/chef/chef-vault/blob/master/THEORY.md), as you'll need that background when implementing chef-vault for an organization. For the first part of your use-case, how to bootstrap a node when using chef-vault, you can use features of `validatorless-bootstrap` when the bootstrapping client (e.g. your workstation) has an administrative key on the chef-server.

One of my chef-vault demos has commands like this:

```
knife bootstrap ec2-52-2-56-144.compute-1.amazonaws.com \
    -N testvault-i-35a4119d   -r 'role[sensu_chefvault]' \
    --bootstrap-vault-json '{"sensu_vault":["rabbitmq"]}'
```

which generates the output like:

```
Creating new client for testvault-i-35a4119d
Creating new node for testvault-i-35a4119d
Connecting to ec2-52-2-56-144.compute-1.amazonaws.com
```

What's going on here is that the `-N testvault-i-35a4119d  -r 'role[sensu_chefvault]'` options create a new `node` and `client` on the chef-server, with runlist 'role[sensu_chefvault]', even if the box in question doesn't exist yet. Further, the the `--bootstrap-vault-json '{"sensu_vault":["rabbitmq"]}'` specifies a vault and item to rekey before continuing with the bootstrap.

When the `knife bootstrap` continues, the newly bootstrapped node will get its private key from the workstation, save it as `/etc/chef/client.pem`, and can use that to unlock the vault items it needs. This works -- it has the key, and the vault has been re-keyed/refreshed from the workstation to allow vault access to testvault-i-35a4119d.

This works great for provisioning/bootstrapping a few nodes from your workstation, but the second part of your question is where things get hairy. In an autoscaling setup with chef-vault you would need to delegate a node in your environment with a large part of the authority that is usually reserved to administrative workstations:

# About chef-vault and autoscaling

Before we go on, bear in mind what my colleague Nathan Cerney says when talking about secrets and Chef:

> Secrets management is about a balance between how much you care about your secrets and how much work you’re willing to do.

Chef-vault is certainly a step up from generic encrypted data_bags; it's an elegant design and a great fit for relatively static sites. But I find that chef-vault is a poor fit to autoscaling situations for the following reasons:

1. You would need some privileged node that has the "keys to the kingdom" to rekey all your vaults for the clients as they come up, which may put some security conscious folks on edge.
2. It's completely on you to write an auditing/authorization framework around that privileged node, as there's nothing in chef-vault that provides that
3. The provisioning process would have to be single-threaded or use some locking around the vault databags or your vaults will get out of sync, or possibly corrupted
4. If you're not willing to delegate rekeying of the vaults to a provisioning node, then you need a human admin to rekeying the vaults with the new search results, and then you have to re-run chef (defeating the whole purpose of autoscaling and automaed provisioning)
5. You're potentially subject to node impersonation attacks, which I describe below
6. Chef-Vault assumes that the set of Chef-Server administrators includes the set of secret administrators, but this is neither true, nor generally desirable. It's not directly related to autoscaling, but is a point to bear in mind when managing secrets for larger organizations - as [noted in this Chef vault blog post](https://www.chef.io/blog/2015/04/28/guest-post-chef-vault-with-large-teams/)

So, now that the bad news is out of the way, how do you manage secrets with Chef in an autoscaling situation?

Noah Kantrowitz has a good 2014 survey of Chef & Secrets landscape here: https://coderanger.net/chef-secrets/ but there have been some interesting developments since then. Three tools you should definitely evaluate are:

- [Hashicorp Vault](https://www.vaultproject.io/) - open-source project from Hashicorp
- [Conjur](https://www.conjur.net/products/secrets-management) - commercial project from Conjur; distributed as an appliance, either an AWS instance, VM or Docker image you deploy internally
- [Conjur Summons](https://blog.conjur.net/introducing-summon) - - summon is an opensource secrets bus that interoperates with other secret backends. I happen to be evaluating it this week, based on this sample cookbook: (https://github.com/conjurinc/summon-chefapi)

Conjur has some features that I find pretty compelling, one that is relevant here is the `hostfactory` feature. Any host in a Conjur environment has to have a key to access its secrets. You can authorize hosts yourself as an admin, or you can assign a `hostfactory` token to an application to delegate host token creation to it, which is what you'd want in an autoscaling setup. I have a [janky demo here](https://github.com/pburkholder/conjur_demo) if you're interested.

Hashicorp Vault has also generated a lot of interest in the last year. The server is open source, and has an h/a mode with Consul. The folks at Bloomberg have a cookbook for management: https://github.com/johnbellone/vault-cookbook

A few other players in the secrets space that Noah didn't mention that I'm aware of:
- KeyWhiz
- AWS KMS
- Sneaker (AWS KMS backend)
- Thycotic
- CyberArk

# General guidance around secrets with Chef

You can think about secrets-management with Chef in a hierarchy:

1. Encrypted data bags(EDBs) with secrets common to all nodes
2. EDBs with secrets shared only among nodes with a common role or run_list
3. Chef-Vault, or EDBs with the EDB decryption key provided from Chef-Vault
  - You may want to take this approach so you can evolve your secrets management from decryption keys coming from chef-vault, to some other service providing the keys.
  - As Nathan pointed out, this approach means you "have an encrypted data bag, protected by a \[key\] that’s stored in an encrypted data bag protected by a secret that’s stored in an encrypted data bag protected by the client’s key," so it's less than ideal
4. Secret storage services: Conjur or Hashicorp Vault or Red October or the like
5. Hardware platforms (HSMs) that are FIPs-compliant and require N approvals and a time-lock

 Each step up makes your infrastructure more complicated, but protects your secrets better. But even Step 1 is immeasurably better than storing secrets in clear-text, and you shouldn't let the perfect be the enemy of the good. Choose an implementation you can realize, then iterate to make it better.


# What is Chef Inc doing to help?

This is some of what's going on here: \[Again, not official\]

1. Providing better documentation and guidance
  - This response is a first draft of new knowledge base article, which will incorporate Noah's blog post, with updates
2. Taking ownership of Chef Vault. The project was originally a Nordstrom project, and they've bequeathed it to us we can continue engineering on it while understanding its limitations
3. Building better examples of integrating Chef and EDBs with third-party secret providers -- this work is already underway on my team, and questions like yours help spur on that effort.

Hope that helps...

--Peter

# <a name="node_impersonation"></a>P.S. What's a node impersonation attack?

Suppose you have:
- a provisioning node that rekeys chef-vault based on updated search results
- an `unprivileged` role that doesn't have any access to interesting secrets
- a `database` role that does have access to interesting secrets

If an attacker gains root on an `unprivileged` node, then s/he can use the client's key to edit the node's role and change it to `database`. Further, s/he can then just kill chef-client runs on that node so it doesn't actually become a `database` node.

Next time the chef-vault is rekeyed, the compromised node (and the attacker) will access to all of the `database` secrets. Ugh.

This same attack works if you have humans doing the rekeying, because, seriously, do you really vet all of the nodes that show up in the search?

This attack against chef-vault will not be possible when [Chef RFC 45](https://github.com/chef/chef-rfc/blob/master/rfc045-node_state_separation.md) is fully implemented, since you can then administratively lock a node's runlist. Pending that implementation, one can also use [Chef-Guard](http://xanzy.io/projects/chef-guard/introduction/overview.html) to filter such attacks, or [Chef Analytics](https://docs.chef.io/analytics.html) to alert on attempts after-the-fact.

*Update 2015.12.11*: Root compromise is not necessary for impersonation.
Filching any `client.pem` _or_ a `validation.pem` will suffice to update or create
client/node objects that match the search criteria, and access vaults.
*Protect your keys*.



Other references:
https://blog.conjur.net/lets-talk-encrypted-data-bags
