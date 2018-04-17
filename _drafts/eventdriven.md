---
layout: post
title: Thinking of an Event Driven Architectur
tags: Event Sourcing Driven Messaging Log
---

With this post I want to start a series of post about event driven architectures. My goal is to sketch an architecture for an event driven microservice architecture using kafka as the central [Log](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying) infrastructure.

In recent years with the advent of [Kafka](https://kafka.apache.org/) and other distributed and by that failsafe event bus systems event driven architectures have become more and more popular. Especially in the area of microservices theses systems start to replace classical REST/HTTP(S) interfaces for inter-service communication. This is because they offer builtin support for downtimes in the service landscape and the different speed of processing of different services.

That is a huge advantage over classical HTTP(S) interfaces as a downtime in one service or a slow service will not compromise the whole service landscape. It also reduces the complexities you usually have when using circuit breakers.

Basic Idea
---

In this section I want to propose a first idea of how to structure the architecture. I also want to explain the advantages and gotchas of the proposes architecture.

The following diagram shows an overview of the architecture:

![EventDrivenArchitecture]({{ "/assets/img/EventDrivenArchitecture.svg" | absolute_url }})

In this architecture I split the whole landscape into different app services. Each app service is made up of:

- Input event queue: This is the queue where all requests that the app service should handle are posted
- Workers: A set of workers that handle all input events
- Materialization DB: A database where the workers put a materialized view of the data as required by the HTTP API of the app service
- HTTP API: An HTTP endpoint for the app service that can for example be used by the UI (of the service), it can read the Materialization DB for presentation of the data and post new events to the input queue to request changes
- Output Event Queue: This is where workers can put events that describe the results of input events, external workers can subscribe this queue



