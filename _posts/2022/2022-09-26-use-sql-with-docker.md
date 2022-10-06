---
layout: post
title: SQL Server 2019 docker image for Sitecore
tags: [docker, sql, Sitecore]
excerpt_separator: <!--more-->
---

Using SQL Server in docker may not be the best idea for production environment, but is certanly usefull for development. With docker we don't have to deal with cleaning up after finished work, neither with unnecessary memory usage when we don't really use the database. Simply spin up containers when you start your work and delete them when you're finished for the day.

<!--more-->

This article was inspired by Sitecore lack of official SQL Server 2019 docker images for development, although it could be usefull to anyone dealing with MS SQL with windows containers. Before Sitecore 10+, if you wanted to use docker, you were forced to build your own images from community supported [repository](https://github.com/Sitecore/docker-images). When in need, or just out of curiosity, one could easily customize what are used for based images, entrypoints, scripts run in the meantime. From versions 10+ Sitecore provides officially supported images, which is awesome, but also makes customization a bit harder.

### Root cause
When my team upgraded from Sitecore 9.2 to 9.3 we decided to go with docker images for development. There are many advantages of that approach, although it required some extra work on our side. One of the first issues was SQL database version used. We used our on&#8209;premise development setup with newest (at that time) SQL Server, but with mentioned earlier repo we could only build 2017 image. The easiest would be to go with it, but if you ever tried to downgrade your database you probably not it's pretty much impossible (if I'm wrong please let me know). 

### To the point
As said, Sitecore doesn't share how the images are built. Luckly with docker we can always access container and check-out what sits inside. First step was to find how to built pure MSSQL2019 image. That microsoft gave us for [free](https://github.com/microsoft/mssql-docker). The next step was to inspect Sitecore provided MSSQL2017 image. Microsoft is here for us again with an awesome VS Code [extensions](https://github.com/microsoft/vscode-docker).