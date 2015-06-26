---
layout: post
title: "That thing to add packaged Gems into your GEM\_PATH"
date: 2015-05-04 09:28:51 -0400
comments: true
categories: 
---

It's `Gem::Specification.reset`  For example, in a Chef cookbook I'm installing the `conjur` package from
[Conjur](https://conjur.net), and need those Gems available to me during
Chef's compilation. So:

```
# Install from a downloaded .deb (or .rpm)
dpkg_package "conjur" do
  source target_path
end.run_action(:install)

# Append those embedded gems in my path:
Gem.path << "/opt/conjur/embedded/lib/ruby/gems/2.1.0"

# And reload the Gem specifications that are in your .path
Gem::Specification.reset()
 ```


It may also be sufficient to do `Gem.clear_paths`, but that also sets your
Gem.paths to `nil` so that doesn't seem right. Haven't tested.


Maybe I should make a PR to alias that to `Gem.reload_paths`


