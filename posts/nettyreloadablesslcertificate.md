---
# title: Netty reloadable SSL certificate.
title: draft
description: Reloading SSL certificate without restarting the server.
date: 2020-10-01
tags:
  - posts-tag
layout: layouts/post.njk
---

Please check the [Netty SSL with CA signed certificates](/posts/casignednettysecuredchat/) before continuing to read this one to learn how set basic netty SSL client server. Once we have basic netty SSL client server working we need to enable server to reload SSL certificates without restarting.

## The theory
Main idea here is to implement a delegate SSL context that would load/reoload SSL context and delegate all methods to it.