---
layout: post
title: "关于Actor模型与游戏编程的一点想法"
date: 2013-12-20 22:28:32 +0800
comments: true
categories: GameDev Actor
---

在读*gevent*的tutorial的时候，一段关于Actor模型的简要描述引起了我的注意：

>Actor模型是一个由于Erlang变得普及的更高层的并发模型。简单的说它的主要思想就是许多个独立的Actor，每个Actor有一个可以从其它Actor接收消息的收件箱。Actor内部的主循环遍历它收到的消息，并根据它期望的行为来采取行动。

看到这段描述突然就想到了七八月份做的MINI项目以及现在正在开发的游戏中，对于游戏的个体之间的交互行为的处理。总结起来，有以下几个关键点：

* 游戏中的各个实体之间的交互行为是很多的，如果单纯使用对象之间相互调用方法的方式来实现交互，代码很快就会耦合成一团乱麻。而Actor模型的消息和收件箱机制，可以很好的解决这个问题，只需要搭建起一个合适的消息中转机制即可。

* 大多数游戏的本质是模拟现实世界，而现实世界中的个体本身就是具有并发性的特点的。我们可以在游戏中串行化地模拟这些个体，从而简化编程过程。但是使用Actor模型，理论上来说可以让我们直接并发地去模拟这些个体。

* Actor模型的“信箱”确实是一个可以处理并发问题的利器。最初在MINI项目中大量使用多线程的时候，就意识到如果可以用队列来处理并发线程之间的所有交互，那么并发问题就可以得到大大简化。队列其实就是一个有序的“信箱”。只是当时并没有想过要把这个思路抽象出来，建立一个统一的框架来处理所有的并发问题。

严谨起见还是去查了一下Wikipedia的定义：

>The actor model in computer science is a mathematical model of concurrent computation that **treats "actors" as the universal primitives** of concurrent digital computation: in response to a message that it receives, an actor can make local decisions, create more actors, send more messages, and determine how to respond to the next message received.

基本概念：

>The Actor model adopts the philosophy that **everything is an actor**. This is similar to the everything is an object philosophy used by some object-oriented programming languages, but differs in that object-oriented software is typically executed sequentially, while the Actor model is inherently concurrent.
An actor is a computational entity that, in response to a message it receives, can concurrently:

>1. send a finite number of messages to other actors;
>2. create a finite number of new actors;
>3. designate the behavior to be used for the next message it receives.

>There is no assumed sequence to the above actions and they could be carried out in parallel.

目前看起来，这个模型用在游戏开发中非常适合。但是真正使用起来会遇到什么困难还是需要实践之后才能知晓。后面的开发中会尝试把Actor模型真正用起来，踩一踩这里面的坑：）