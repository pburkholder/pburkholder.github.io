---
layout: post
title: "Chef custom resources: here's an easy example"
date: 2016-03-04 10:59:45 -0500
comments: true
categories:
---
# Chef custom resources are easy

Purpose: Demonstrate a simple custom resource cookbook with tests.

When it comes to programming, I'm an inductive learner. To me, a working example is worth a 1000 lines of documentation. The Chef Custom Resource model introduced with the release of 12.4 offers a way to write reusable _resources_ that's far simpler than the Chef LWRP and HWRP models (although those are still fully supported). And there's strong documentation that came with it, to wit:

* Docs: https://docs.chef.io/custom_resources.html
* Slides: https://docs.chef.io/decks/custom_resources.html
* Webinar: https://www.chef.io/webinars/?commid=175693
* Webinar FAQ: https://www.chef.io/blog/2015/11/06/custom-resources-in-chef-client-12-5/

However, if you just want to see trivially simple example from which you can grow, then the `motd_cr` cookbook is for you: [https://github.com/pburkholder/motd_custom_resource](https://github.com/pburkholder/motd_custom_resource).

It's a _library_ cookbook, meaning there's no default recipe, and it adds a `motd` resource to Chef when you add `depends "motd_cr"` to your `metadata.rb` any external cookbook.

The relevant custom resource code is in `resources/motd.rb`, and it's merely this:

```
resource_name :motd
property :message, kind_of: String, name_property: true

action :create do
  file '/etc/motd' do
    content "#{message}\n"
    mode '0644'
  end
end
```

Yes, it's that's short.

The example cookbooks for testing are in `tests/fixtures/cookbooks/test`.  Basically  you use the resource like this:

```
motd 'Welcome to my message of the day'
```

And working tests are in `tests/` and `spec/`. Running the kitchen and chefspec tests should convince you it works

I hope this helps you write your own custom resources complete with ChefSpec and Test-Kitchen tests

-- Peter

# TESTS

## Integration testing

```
kitchen test
```

(If you want to use this with the ultra-fast kitchen-dokken framework, do this first:)

```
export KITCHEN_LOCAL_YAML=.kitchen.dokken.yml
eval $(docker-machine env default)
```

## Unit testing

ChefSpec and custom resource:
```
rspec spec/unit/recipes/default_spec.rb
```

# Anticipated Questions:

## What about `load_current_value` and `converge_by`?

These aren't generally needed if your custom resource comprises a collection of native chef resources, since each of them will compare the current state to the desired state and take no action when none is needed.

When you are building new resources from Ruby code then you'll need the above methods so your resource follows the test-and-repair approach (idempotency).

## Why doesn't `--why-run` work as expected?

It's a bug: https://github.com/chef/chef/issues/4537

## Doesn't the resource need an `action :destroy`?

Sure - fork this and write it. It's a learning exercise, not something I expect anyone to use.

## What about poise-style testing?

Oh that -- to minimize confusion I've moved it to the `poise` tag and `poise-dev` branch of this repo.  Look there for more info.
