---
layout: post
title: Minimal Micro Services with Golang
tags: go golang single page application spa web server http alpine docker
---

Recently while working on a java project I realised how complicated and slow everything seemed to be. I also started looking into the GO language and thought of doing a post about minimalism while programming.


## Minimalism with go

Recently I started learning golang and did some implementation work with it. I really like the language. There are potentially many reasons to like the language (as there are reasons to like other languages). Some follow. It strikes a good balance between verbosity and clarity by e.g. ommiting some parentheses and all semicolons. Writing it is just simpler even in a basic editor. It is compiled down to machine code which makes it very fast. Despite this go is simpler (compare to e.g. rust) and safer (compared to C and C++) than most other binary compiled languages. You do not need to care too much about memory management because it has garbage collection.

Therefore my strategy to minimise apps and deployments in this post will be:

- Code webservice in go
- Compile webservice without dependencies
- Run webservice in minimal docker container

## The runtime dilemma

Because go is compiled to machine code it does not need a runtime like java or .Net. That has several advantages. 

The java runtime is quite large. The image for `openjdk:11-jre-slim` is over 200 MB large. The .netcore runtime image `mcr.microsoft.com/dotnet/core/runtime:3.1` is almost 200 MB large. That is roughly ten times the size of most linux distributions today. Go binaries can potentially be very small. A simple webserver has less than 10 MB.

Distributing a go app is very easy because it basically is only one binary. You do not need to install any dependencies. One of my recent experiences with this is installing terraform. Just download the binary and execute it.

The java runtime has a very tedious property. It reserves memory for your app upfront. By default it will reserve a very large amount. From experience with spring boot services its between 500MB and 1GB or even more. Thats a lot of RAM for a little microservice. You can tweek this value down but it is neither easy nor straightforward to get this right. This is especially bad when many java apps should run in docker containers on a single host.

## Writing a SPA server

As an example I wrote a minimal SPA server. You can find it on github

    https://github.com/Useurmind/spas

The most complex part of this server really is retrieving the options from the different sources commandline, environment and configfile (perhaps I should use a library for this). The rest is plain go code using the base libs (mainly http) to serve the files of the single page app.

To bring the numbers of the previous section into perspective let me tell you some numbers about this app. As of writing this the spa server has a compiled size of 7.5 MB. If you run it serving a little spa app it takes up 1.5 MB of RAM when idle.

Clearly it achieves the goal of a low resource consumption.

## Why not just nginx

Nginx is a very customizable webserver. It is also very complex to configure it correctly and to test that it does exactly what you want. The usecase of a spa server is much more limited than what nginx can do. And I personally think you should not use something so big to handle such a small problem. 

That basically boils down to the same discussion regarding containers and alpine where less code means less attack surface.

## A minimal docker container

The first question to answer here is which base image to use. Because of my familarity with ubuntu it was a first candidate.

But there is this ongoing debate about minimal docker images. Minimal docker images reduce the attack surface by removing code that could be attacked. Also small docker images reduce the network, cpu, disk, and memory consumption.

## Docker base image sizes

Here is a comparisson of a few distros at the time of writing.

| distro | tag | size |
| --- | --- | --- |
| alpine | latest | 5.6MB |
| ubuntu | latest | 64.2MB |
| debian | stable-slim | 69.2MB |
| debian | latest | 114MB |

You see that an alpine image is much much smaller than the usual linux images.

## FROM alpine

Alpine is one of the smallest linux distributions you can use for containers. It comes with a shell and the package manager `apk` that allows you to get basic tooling easily e.g. curl or bash. But there are a lot of differences between a debian based system and alpine. The biggest difference is that alpine uses a completely different c library [musl libc](https://musl.libc.org/). This can lead to differences between the native programs in alpine and other distributions. You also have to rebuild C programs against alpine to make them work on it.

## FROM scratch

In addition you can build your image from scratch which means no distribution is used as the base image. You can read more about the `FROM scratch` image in the [official docs](https://hub.docker.com/_/scratch) or on [this blog post](https://www.mgasch.com/post/scratch/).

Basically the scratch image gives you a default root file system provided by the container runtime with very minimal files and folders. You don't have a shell installed and no package manager. This implies that you cannot just log into your scratch image from remote to see what is going on. It is also hard to install dependencies for complex applications.

## Choosing a base image for go

In our case most drawbacks of `FROM alpine` and `FROM scratch` do not apply. We are packaging a go application whithout any other dependencies using CGO_ENABLED. 

Nevertheless `FROM scratch` has a drawback that can be a problem in some circumstances: it does not have a shell and other tooling installed. That means you can not just log into the container remotely to debug what is going on. When using such a container in production you would have to consider if operations personal want to log into it at some point. Or at least make sure that you have a way to analyze what is going on inside the container.

Therefore for development tasks it does not makes sense to use a `FROM scratch` image. It just makes your life harder than it needs to be. Alpine is a good medium choice because it is small and you can install some basic tooling for debuging problems directly in the container. Small means your development environment will stay fast. And you will need the tooling at some point.

For production purposes I would say it depends. Operations personnel can always log into the docker host if need be. From there you can leverage tools to analyse the different containers like `docker stats` or `docker top`. You should also have some distributed logging and monitoring installed. If you can manage to install them and still using a `FROM scratch` image it should be fine. If not you probably need to switch to something that supports those tools.
