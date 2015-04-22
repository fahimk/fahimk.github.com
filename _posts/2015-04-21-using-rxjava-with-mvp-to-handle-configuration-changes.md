---
layout: post
title: "Using RxJava with MVP to handle configuration changes"
description: ""
category: 
tags: []
---
{% include JB/setup %}
###the issue
Configuration changes with long running tasks (network calls, database operations) often results in a bad user experience.

![array items not saved]({{ site.url }}/assets/bad0.gif)
![array items and state not saved]({{ site.url }}/assets/bad1.gif)