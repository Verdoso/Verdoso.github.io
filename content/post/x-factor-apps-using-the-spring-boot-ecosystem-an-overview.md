---
date: 2017-10-08T00:00:00+02:00
tags:
- Docker
- Spring Boot
- Java
- Development
featured_image: ''
title: X-Factor Apps using the Spring Boot ecosystem, an overview

---
### Summary

Nowadays, there are tons of literature and virtual ink written about how to develop for the cloud and make your applications Internet-scale. But how about those of us where the scale of our problems do not fit such model? Well, even if you are not developing for the cloud and you don’t need to go full scale with the [Twelve-Factor App](https://12factor.net/) recommendations, there are still many benefits that can still be enjoyed by sticking to the main principles behind them. They are based on good practices to facilitate management & deployment of applications, a requirement if you have to develop software as a service, but they still contain sound advice that can be applied in any circumstance. Quoting those basic principles, an app should:

1\. Use declarative formats for setup automation, to minimize time and cost for new developers joining the project;

2\. Have a clean contract with the underlying operating system, offering maximum portability between execution environments;

3\. Be suitable for deployment on modern cloud platforms, obviating the need for servers and systems administration;

4\. Minimize divergence between development and production, enabling continuous deployment for maximum agility;

5\. Be able to scale up without significant changes to tooling, architecture, or development practices.

In this entry, I’ll outline a set of development practices, using [Spring Boot](https://projects.spring.io/spring-boot/) & co, to get you closer to these goals.

### Basic steps

We are discussing here the development of Spring Boot applications, so you might want to start by using Spring Boot to [develop a packaged application](https://docs.spring.io/spring-boot/docs/current/reference/html/using-boot-running-your-application.html) that requires no external container to be run. That provides the benefit of requiring no external servlet container setup & configuration, which is usually a hassle when creating new environments from scratch for new team members or new deployment environments.

On top of that, in order to minimize hardcoded configurations, that would cause issues when deploying the packaged application in different environments, you could then [externalize the configuration](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html) values of your application and then move a step further and [centralize it](https://spring.io/guides/gs/centralized-configuration/), using a [Spring Cloud Config](https://cloud.spring.io/spring-cloud-config/) Server. If the properties you want to externalize contain sensitive data, like database connection strings or passwords, don’t forget that you can [encrypt the sensitive values](https://cloud.spring.io/spring-cloud-config/single/spring-cloud-config.html#_encryption_and_decryption) in the properties so you don’t have to expose them in plain text in your centralized configuration

With the above steps, you have mainly covered principles 1 and 4: The packaged app can be built, packaged and deployed automatically, from declarative files, and you can use the same artifact in all environments thanks to having the environment-specific properties externalized in a centralized server. That alone, is a big move forward compared to the once very common, and still present: “_These are the instructions to set up the application in your environment and these are the properties or files you have to write so the app runs (different for each environment)_”.

### Becoming a shepherd

Now that you can run the application more easily in different environments, it is far easier to set up new servers, so you feel the need to start [treating the servers more like cattle rather than like pets](http://cloudscaling.com/blog/cloud-computing/the-history-of-pets-vs-cattle/). That is, you want to be able to spawn new servers with the new app without having to worry about the specific details of the servers, like IPs, ports, names. But that poses new challenges: How are clients going to call an app if they don’t know the IP of the server where it is deployed? Well, one solution is using a balancer with a well-known name, at least known to the clients, that redirects the traffic to the appropriate server(s). But that simply pushes the problem one layer because, now how does the balancer know the IP of the servers where the app is deployed?

Previously, the answer to this was “someone configures the balancer with the addresses of the servers where the app is deployed”. But now we want cattle, not pets, hence no fixed servers where the application is deployed. What can we do now? The answer is using a discovery service where apps register upon startup. This kind of services can be used to keep a list of apps/servers that the balancer can query to configure itself. For example, one can use [HashiCorp Consul](https://www.consul.io/) and [Spring Cloud Consul](https://cloud.spring.io/spring-cloud-consul/) so the Spring Boot applications register automatically when they start. Moreover, the URL of the discovery service can be an externalized property so the app can register itself in a different server depending on the environment (through spring active profiles and a different url in the externalized properties).

Once you have your apps registered in Consul, you can use [Consul Template](https://github.com/hashicorp/consul-template) to query the active apps and the data of the servers hosting each of them and then [update automatically the configuration of your balancer](https://www.hashicorp.com/blog/introducing-consul-template/)(be it Apache, Ngingx,HAProxy or others).

If you play your cards appropriately, you can configure it so when you start an instance of an app named myApp in the environment myEnv, you can then access it behind the name myApp.myEnv.myservices.mydomain.com. Moreover, if you start more than one server in the same environment, the balancer will distribute the traffic among them. Cool, huh?

### The toppings

Don’t forget that once your servers are no longer pets, you’ll need to centralize their logs, as otherwise accessing them or even just keeping them might be a headache. You could use for that a logging platform like [GrayLog](https://www.graylog.org/) or [Elastic Stack](https://www.elastic.co/webinars/introduction-elk-stack) (ELK) or one of the various Logging-As-A-Service cloud based providers, depending on your needs and budget.

While you are at it, you can add some metrics & monitoring to your servers/services with something like [Graphite](http://graphiteapp.org/) or one of the various monitoring-as-a-service offerings, cloud based or on premise.

As you can see, just to test most of these things one would need to install many products in various servers, so it then becomes a good idea to start experimenting with containers so you can do it more easily. There are docker images of pretty much all of the services I linked above and you can easily [create your own docker files to deploy your Spring Boot applications](https://spring.io/guides/gs/spring-boot-docker/), so you don’t need a whole datacenter to test all the different pieces working together.

And that’s it, for now. I know this is just an outline and I did not provide much detail into how to actually implement any of those steps in a real application, but that would require not just one entry but a full series of entries. I always prefer to start by setting the end goal clearly, so all the intermediate pieces make sense and you just don’t follow one technique after another wondering “why am I doing this?”. So now that the destination is clear, I might add more entries describing some of the steps in more detail.

I hope you enjoyed it, happy coding!

D.