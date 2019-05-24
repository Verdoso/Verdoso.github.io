---
date: 2018-04-10T18:00:36+02:00
tags:
- Elasticsearch
- Consul
- DevOps
featured_image: ''
title: 'Quick tip: Elasticsearch + Consul + health check + basic authentication'

---
Recently, I’ve been installing the “[Elastic Stack](https://www.elastic.co/products)”, previously known as ELK, and one of the steps took me a while to figure it out, due to outdated documentation and missing pieces in the examples for the specific syntax that you need to get right, so here it comes:

Imagine you have already gone through the [Elastic Stack quick start](https://www.elastic.co/start) and you have it secured with basic authentication, either directly through the configuration or through the recommended [X-Pack plugin](https://www.elastic.co/products/x-pack).

Imagine as well that you have a [Consul](https://www.consul.io/) cluster where you register your services so you can do some monitoring and load-balancing (I’m using [Fabio](https://fabiolb.net/)for that and I’m truly happy with it).

Now you want to register your [Elasticsearch](https://www.elastic.co/products/elasticsearch) service in your consul cluster, in order to do so, you will probably want to register the [Elasticsearch](https://www.elastic.co/products/elasticsearch) application [as a service](https://www.consul.io/docs/agent/services.html) with a configuration file, in the consul.d directory, like this one:

{{< gist Verdoso 33a1827eafe90e570f8d843913261913>}}

So simple right? But then you want more, so you decide to add a check so you can detect when the node is offline, so you go and add the check section to the configuration, like this (note that the check uses a simple HTTP check against the port because we don’t want to check the cluster health to decide about this specific node’s health):

{{< gist Verdoso b7895c1dcd822630fbe7bccf02e8343d>}}

And kaboum! Consul says your node is always offline. Why? Because the Elasticsearch service is protected and the check is returning a “401 Unauthorized” response. So how do you add the proper credentials for the check? Well, there are many issues open in Consul about how to add headers to the HTTP checks (see for example [#1896](https://github.com/hashicorp/consul/issues/1896), [#1184](https://github.com/hashicorp/consul/issues/1184) and [#3107](https://github.com/hashicorp/consul/pull/3107)) , but after quite a bit of trial & error, this is the syntax that works:

{{< gist Verdoso 4f7012bfe47ac2d82524bf98fb4dfd8b>}}

Note that “_T2hZZWFoOllvdXRob3VnaHR0aGF0d2FzbXlwYXNzd29yZD8=_” are the combined credentials (username:password) encoded using Base64. You can get your own version using any program that is able to produce those headers.

Note also that the credentials can be easily decoded (they are simply encoded, not encrypted) so make sure the file is well protected. Creating a user in Elasticsearch with just the most basic permissions to check the node status and nothing else, and using that would also be worth considering.

In any case, update the Consul service configuration , reload Consul and the check should work fine and your Elasticsearch node should be green in your Consul console.

That was all for today. It took me a while to put figure it out, so I thought I could save someone else, or even a future me ;), the trouble.

Happy deploying!

D.