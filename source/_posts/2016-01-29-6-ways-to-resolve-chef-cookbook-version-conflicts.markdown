---
layout: post
title: "6 ways to resolve Chef cookbook version conflicts"
date: 2016-01-29 19:06:58 -0500
comments: true
categories:
---

Six ways to resolve Chef cookbook version conflicts

## Problem

_From one of my customers:_

Our `appdev` role now depends on the `perforce` cookbook from Supermarket. The `perforce` cookbook's metadata specifies the dependency `depends windows, '~> 1.38'`

Problem is, we've pinned our five environments to windows 1.36.1, and we're not in a position to upgrade to 1.38.0 just now.

We're using a current version of chef-client and chef-server. How can we include Perforce in our appdev role without inflicting too much pain on ourselves?

## Solution

There's no perfect solution for this, as they all have some tradeoffs, so you'll need to decide with path works for you.

This answer refers to the `perforce` and `windows` cookbooks per the original answer, but substitute in whatever cookbook constraints you are working under.

### Preliminaries:

Before we address the various approaches, let's review cookbook dependency management.  First, cookbook dependencies for Chef 11/12 can be specified in any of four places:

- The cookbook's `metadata.rb`
- The node's environment, e.g. in an `environment.json`
- The node's role, e.g. in a `role.json`.  Note: Many chef user's regard this use of roles as an anti-pattern.
- The node's runlist, e.g. `"run_list": [ "recipe[foo::bar]@0.1.0" ]` Note: This is largely undocumented and unsupported. Do not use.

Unlike `attributes`, there is no precedence for cookbook versions. Instead the chef-server's `depsolver` will find the newest cookbook version to satisfy all the constraints, or throw a 412 error. You may want to read https://getchef.zendesk.com/hc/en-us/articles/204381030-Troubleshoot-Cookbook-Dependency-Issues if you're trying to identify what cookbook constraint is breaking your chef run.

For newer Chef 12 (server & client) installs, one can use Policyfiles instead of the above approach to specify cookbook dependencies.

### Options

In brief, here are the options you have:

1. Use an older version of the offending cookbook
1. Fork the offending cookbook and yank the stuff that requires a version newer than you want to use. (Or don't use the perforce cookbook at all, write one that does only what you need)
1. Bump to Windows 1.38.0 everywhere
1. Move the cookbook constraints from environments to cookbooks
1. Create a microenvironment where you could pin to Windows 1.38.0
1. Use policyfiles for the nodes in question

In more detail:

#### Use an older version of the offending cookbook

If you really want to keep your cookbooks as they are on the Supermarket, this may be an approach that works for you. Or the cookbook author may be able to respond to an issue submission or a pull-request if you have fix that works with wider cookbook version constraints.

#### Fork the offending cookbook and yank the stuff that requires a version newer than you want to use. (Or don't use the perforce cookbook at all, write one that does only what you need)

In this case you are maintaining your own code, but that may be more maintainable than using the Supermarket cookbook

#### Bump to Windows 1.38.0 everywhere

If you have a solid cookbook development and testing pipeline, this is arguably the best solution as you want to keep your code on 'master' as much as possible. However, if your pipeline is still in the works, or you work under change management constraints, you may not have this option.

#### Move the cookbook constraints from environments to cookbooks

However, if you have lots of cookbooks, or have a workflow that leverages `berks apply` or otherwise is tuned to dependency constraints in environments, then this option is a non-starter.

#### Create a `microenvironment` where you could pin to 1.38.0

I don't know if `microenvironment` is a common term for this, but if your currently have, say, `sandbox`, `dev`, `qa`, `preprod` and `prod` environments, then you may be okay adding just an `appdev` environment. However, if your role is ubiquitous, then you might have to go with five new envs: `sandbox_appdev`, `dev_appdev`, etc.

#### Use policyfiles for the nodes in question

Chef Policyfile was developed with just this sort of use-case in mind. You could apply a Policy for the `appdev` nodes in your orgs while leaving all the other nodes in the Chef 11/12 constraint solution world. The Chef Server 12.2.0 and higher can support both types of nodes just fine, so you might want to use this as an opportunity to start working with Policyfile for your applications and nodes.

See also: https://docs.chef.io/policy.html and https://www.chef.io/blog/2015/08/18/policyfiles-a-guided-tour/ and https://github.com/chef/chef-dk/blob/master/POLICYFILE_README.md

# Related Articles

[Chef Docs on Cookbook Versions](https://docs.chef.io/cookbook_versions.html)


[Troubleshoot Cookbook Dependency Issues](https://getchef.zendesk.com/hc/en-us/articles/204381030-Troubleshoot-Cookbook-Dependency-Issues)

# Update

In this case, the problem was solved by the `perforce` cookbook getting an update to relax the constraints on the versions of the windows cookbook, a variation of number 1, above.
