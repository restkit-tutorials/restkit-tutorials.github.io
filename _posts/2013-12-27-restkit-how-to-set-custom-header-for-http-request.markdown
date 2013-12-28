---
layout: post
title: "RestKit - How to set custom header for HTTP request"
date: 2013-12-27 10:18:58 +0200
published: false
---
There are 2 different approaches to solve this problem. We'll go thru each of them and describe when to use them.

You can set custom header:

* on `RKObjectManager` level
* on `RKObjectRequestOperation` level via `RKHTTPRequestOperation`