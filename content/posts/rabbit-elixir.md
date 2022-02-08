---
title: "RabbitMQ messaging with Elixir"
date: 2020-03-20T21:20:07+01:00
draft: false
toc: false
images:
tags: 
  - rabbit
  - elixir
---

[RabbitMQ](https://www.rabbitmq.com/) is one of the most popular open source message brokers. There are good reasons for it: it's fast, reliable, well supported and documented. The project's tagline is "Messaging that just works", which is true in my experience, but there are scenarios when it can be dfficult to implement a messaging client correctly. In this post I'll describe a messaging client in Elixir that acts both as a publisher and consumer of messages.

It may seem strange to force both operations, publishing and consuming, into one service. Normally, they are a completely separate with a bunch of services publishing messages and then a pool of workers consuming those messages. These 'work pipelines' are common when processing streaming data. But there are valid use-cases when you'd want both operations in one service. For example, an authentication service might publish login events to a queue and concurrently consume provisioning events from another queue (user created/deleted/blocked/...). Or a service wants to use queueing internally as a way to implement back pressure and retries, so it publishes incoming requests as messages and consumes them later when ready.

Let's say we need to implement Rabbit integration in an Elixir service:

1. it has to support publishing messages to a queue,
2. it has to support consuming of messages from a queue.

It makes sense to start with the obvious: use a library that does the heavy lifting for us such as implementation of [AMQP](https://en.wikipedia.org/wiki/Advanced_Message_Queuing_Protocol). But even having a library like that we still have to decide how to approach the implementation. Should we have one connection or pool of connections? One channel per connection or multiple channels per connection?

Note that I'll cover only the bits around RabbitMQ integration and leave out the details around Elixir/Phoenix apps, configuration/releases, etc. 

To begin with, we need to include a library that knows how to work with AMQP and RabbitMQ. Let's pick [pma/amqp](https://github.com/pma/amqp) - to include it in an Elixir project, add following to the dependency list in `mix.exs`:

```elixir
def deps do
  [
    ...
    {:amqp, "~> 1.3.1"}
  ]
end
```

The library provides building blocks needed to work with RabbitMQ:

1. connections,
2. channels,
3. exchanges,
4. queues,
5. and utilities to use them easily.

We'll take that, thank you, and build our service on top of it.

First, a small note regarding connections and channels. A RabbitMQ 'connection' is what you'd expect it to be: a TCP connection to the rabbit instance with defined address and port, initial authentication and so on. A 'channel' is a lightweight abstraction for performing specific tasks like declaring an exchange or subscribe to a queue. Single connection can have multiple channels and multiplex various operations via those channels. Channels can be short-lived just for a single operation or long-lived for a series of operations. Example use case for long-lived channel is consuming messages from a queue.

While it's possible to do everything using just one connection (with multiple channels), various sources and recommendations regarding RabbitMQ clients suggest to have separate connections for long lived channels. Or in other words, publishing should be separate from consuming. The major advantage here is that if consuming fails, publishing still works and vice versa. 

With that in mind we should be able to build a working solution with one connection to publish messages to an exchange and another to consume messages from a queue. 

