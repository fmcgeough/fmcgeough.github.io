---
layout: post
title: Elixir, ex_aws Library, Temporary Security Tokens
date: 2017-04-12 08:53:13
description: Features of the ex_aws Elixir library
categories: Elixir
tags: Elixir
---

During a development effort where I had to use AWS API's to extract information
about our EC2 instances, the network powers-that-be wanted to just grant
me temporary AWS credentials. I wanted to use [ex_aws](https://hex.pm/packages/ex_aws)
since its by far the most popular Elixir library for using AWS' numerous
API's but I hadn't used it with temporary security tokens before. It took
a while to figure this out so I figured I'd write a short post about what I
did.

I got the temporary keys which include a access key, a secret access key,
and a security token. I set all these as environment variables for my shell.
In my Elixir code I configured ex_aws using the following :

```
    config :ex_aws,
      debug_requests: true,
      access_key_id: [{:system, "AWS_ACCESS_KEY_ID"}, :instance_role],
      secret_access_key: [{:system, "AWS_SECRET_ACCESS_KEY"}, :instance_role],
      security_token: [{:system, "AWS_SECURITY_TOKEN"}, :instance_role],
      region: "us-east-1"
```

It seems very straightforward now looking at it but it still took me a while
to get this to work.
