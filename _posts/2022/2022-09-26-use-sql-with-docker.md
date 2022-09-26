---
layout: post
title: SQL Server with docker
tags: [docker, sql]
excerpt_separator: <!--more-->
---

There is no real application without persistent storage, most likely your app will need to use a database. In production environment you don't want to handle its maintenance, thus it's better to get it from provider of choice. On the other hand installing new database or cleaning up after finished project takes time and effort. Here docker gives you a hand!
<!--more-->

Today I want to show you how you can easily build your own SQL Server Developer 2019 image for windows containers.
