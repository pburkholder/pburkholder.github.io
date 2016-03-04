---
layout: post
title: "Lambda to defer chef code evaluation"
date: 2016-02-03 11:01:43 -0500
comments: true
categories:
---

**Draft Note**

TKTK: needs explanation

Don't do this, for examplar purposes only:

```
cookbook_file '/etc/chef/encrypted_data_bag_secret' do
  source 'encrypted_data_bag_secret'
end

# create an 'aws' lambda to call during converge phase
aws = lambda do
  data_bag_item(
    'encrypted', 'aws', IO.read('/etc/chef/encrypted_data_bag_secret')
  )
end

template '/home/ubuntu/.s3cfg' do
  source 's3cfg.erb'
  owner 'root'
  group 'root'
  mode 00744
  variables lazy {
    {
      aws_secret_key: aws.call['aws_secret_key'],
      aws_access_key: aws.call['aws_access_key']
    }
  }
end
```
