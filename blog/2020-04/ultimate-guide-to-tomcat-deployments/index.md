---
title: The ultimate guide to Tomcat deployments
description: In this blog post we'll create a secure, highly available, load balanced Tomcat cluster with zero downtime deployments
author: matthew.casperson@octopus.com
visibility: private
published: 2999-01-01
metaImage:
bannerImage:
tags:
 - Octopus
---

Continuous integration and delivery (CI/CD) is a common goal for DevOps teams to implement to reduce the cost and increase the agility of software teams. But the CI/CD pipeline is far more than simply testing, compiling and deploying applications. A robust CI/CD pipeline needs to address a number of concerns such as:

* High availability (HA)
* Multiple environments
* Zero downtime deployments
* Database migrations
* Load balancers
* HTTPS and certificate management
* Feature branch deployments
* Smoke testing
* Rollback strategies

How these goals are implemented depends on the type of software being deployed. In this blog post we'll look at how to create a CI/CD pipeline deploying Java applications to Tomcat. We'll them build a supporting infrastructure stack including the Apache web server for load balancing, PostgreSQL for the database, Keepalived for highly available load balancers, and Octopus orchestrating the deployments.

## A note on the PostgreSQL server

This blog post assumes that the PostgreSQL database is already deployed in a highly available configuration. For more information on how to deploy PostgreSQL, refer to the [documentation](https://www.postgresql.org/docs/current/high-availability.html).

The instructions in that blog post can be followed with a single PostgreSQL instance, with the understanding that the database represents a single point of failure.

## Implementing HA in Tomcat

When talking about HA, it is important to understand exactly what components of a platform need to be managed to address the unavailability of an individual Tomcat instance, for example when an instance is unavailable due to routine maintenance or a hardware failure.

For the purpose of this post we will create infrastructure that allows a traditional stateful Java servlet application to continue to operate when an individual Tomcat server is no longer available. In practical terms this means that the application session state will persist and be available when the server originally hosting the session is no longer available.

As a brief recap, Java servlet application can save data against a `HttpSession` instance which is then available across requests. In the (naïve, as it does not deal with race conditions) example below we have a simple counter variable that is incremented with each request to the page. This demonstrates how information can be persisted across individual requests made by a web browser:

```Java
@RequestMapping("/pageCount")
public String index(final HttpSession httpSession) {
    httpSession.setAttribute("pageCount", ObjectUtils.defaultIfNull((Integer)httpSession.getAttribute("pageCount"), 0) + 1);
    return (Integer)httpSession.getAttribute("pageCount");
}
```

The session state is held in memory on an individual server. If that server is no longer available, the session data is also lost. For a trivial example like a page count, this is not important. But it is not uncommon for more critical functionality to rely on the session state. For example, a shopping card may hold the list of items for purchase in session state, and losing that information may result in a lost sale.

Therefor, to maintain high availability, the session state needs to be duplicated so it can be recovered if a server goes offline.

Tomcat offers three solutions to enable session replication:

1. Using session persistence, and saving the session to a shared file system (PersistenceManager + FileStore)
2. Using session persistence, and saving the session to a shared database (PersistenceManager + JDBCStore)
3. Using in-memory-replication, using the SimpleTcpCluster that ships with Tomcat (lib/catalina-tribes.jar + lib/catalina-ha.jar)

Because our infrastructure stack already assumes a highly available database, we'll implement option two. This is arguably the simplest solution for us, as we do not have to implement any special networking, and can reuse an existing database. However, this solution does introduce a delay between when the session state is modified and when it is persisted to the database. This delay introduces a window during which data may be lost in the case of hardware or network failure. Schedule maintenance tasks are supported though, as any session data will be written to the database when Tomcat is shutdown, allowing us to patch the operating system or update Tomcat itself safely.

We noted that the example code above is naïve as it does not deal with the fact that the session cache is not thread safe. Even this simple example is subject to race conditions that may result in the page count being incorrect. The solution to this problem is to use the traditional thread locks and synchronization features available in Java, but these features are only valid within a single JVM. This means we must ensure that client requests are always directed to a single Tomcat instance, which in turn means that only one Tomcat instance contains the single, authoritative copy of the session state, which can then ensure consistency via thread locks and synchronization. This is achieved with sticky sessions.

Sticky sessions provide a way for client requests to be inspected by a load balancer and then directed to one web server in a cluster. By default in a Java servlet application a client is identified by a `JSESSIONID` cookie that is sent by the web browser and inspected by the load balancer to identify the Tomcat instance that holds the session, and then by the Tomcat server to associate a request with an existing session.

In summary, our HA Tomcat solution:

* Persists session state to a shared database.
* Implements sticky sessions to direct client requests to a single Tomcat instance.
* Supports routine maintenance by persisting session state when Tomcat is shutdown.
* Has a small window during which a hardware or network failure may result in lost data.

## Implementing HA load balancers

To ensure that network requests are distributed amongst multiple Tomcat instances and not directed to an offline instance we need to implement a load balancer solution. These load balancers sit in front of the Tomcat instances and direct network requests to those instances that are available.

Many load balancer solutions exist that can perform this role, but for this blog post we'll use the Apache web server with the mod_jk plugin. Apache will provide the networking functionality, while mod_jk will distribute traffic to multiple Tomcat instances, implementing sticky sessions to direct a client to the same backend server for each request.

In order to maintain high availability, we need at least two load balancers. But how do we then split a single incoming network connection across two load balancers in a reliable manner? This is where Keepalived comes in.

Keepalived is a Linux service run across multiple instances and picks a single master instance from the pool of healthy instances. Keepalive is quite flexible when it comes to determining what that master instance does, but in our scenario we will use Keepalived to assign a virtual, floating IP address to the instance that assumes the master role. This means our incoming network traffic will be sent to a floating IP address that is assigned to a healthy load balancer, and the load balancer then forwards the traffic to the Tomcat instances. Should one of the load balancers be taken offline, Keepalived will ensure that the remaining load balancer is assigned the floating IP address.

In summary, our HA load balancing solution:

* Implements Apache with the mod_jk plugin to direct traffic to the Tomcat instances.
* Implements Keepalived to ensure one load balancer has a floating IP address assigned to it.

## Implementing zero downtime deployments