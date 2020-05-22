---
defaults:
  # _posts
  - scope:
      path: ""
      type: posts
    values:
      layout: single
      author_profile: true
      read_time: true
      comments: true
      share: true
      related: true
---

The Challenge
=============

I recently ran into an interesting challenge.  My company maintains suites of automated API tests in a [Maven][https://maven.apache.org] project, which
we run using [JUnit][https://junit.org/junit4/].  We've been considering modernizing our Continuous Integration (CI) implementation and noticed that modern
CI systems now commonly use [Docker][https://www.docker.com/] containers to implement their steps.

As I considered how I would containerize our API test suites, I made the decision that I didn't want to abandon the functionality that Maven provides.
The [Surefire Plugin][https://maven.apache.org/surefire/maven-surefire-plugin/], as one example, provides the framework of our entire test suite execution.
Our tests, developed over the course of a few years, made assumptions that this would be their execution environment, and we use the test suite reporting,
flow control, and test selection functionality that's part of the plugin.  We also rely on the parallel execution it supports.  I didn't want to replace
any of that functionality...instead, I wanted this all to happen quickly and easily in a Docker container.

As I began to build a container with our build, another use case presented itself to me:  Our platform runs both in a public cloud environment and in
on-premise installations.  Some of our test suites are useful for validating successfully-installed software on-prem, and it would be great to be able
to run this in those contexts.  I realized two problems with Maven's practice of assuming a connected environment when it executes:

- By default, my container downloads all of its dependencies each time it executes.  This wastes time and bandwidth.  When this container is used in an
  on-prem context, I would like to avoid the need for it to contact my private artifact repository to download dependencies at all.
- Depending on how dependency versions are specified (or not specified) in the POM for this project, it's possible a dependency could change between the
  time of building my container and its execution.

I decided it would be better to pre-package any needed dependencies inside the container so that each image is complete and doesn't have any external
dependencies (other than the services it tests).

Maven's Offline Mode
====================


