---
layout: post
title: "RxJava with MVP to handle configuration changes"
description: ""
category: 
tags: []
---
{% include JB/setup %}
####intro
Configuration changes with long running tasks (network calls, database operations) often results in a bad user experience. Here is a simple app that gets a list of artists based on a search. Even though the network call is maintained, through the orientation, the progress circle disappears. There is no indication that the results are still coming in, but they do eventually. 

![array items and state not saved]({{ site.url }}/assets/bad1.gif)

If the callback isn't persistant through the lifcycle changes, then you could end up with just a blank search page on every rotation.

Since the activity is being recreated on the orientation change, we need to keep track of both the network data that is coming in and the current "state" of the activity. State in this case refers to what is currently being displayed to the user.

####options
Up until now, I've used robospice which gave the ability to get pending listeners after a rotation. However, this solution excludes the state so that would have to be handled independently. 

I stumbled upon two frameworks that both take this issue on directly:

https://github.com/doridori/Dynamo

https://github.com/sockeqwe/mosby

Both have a similar style, where there is a component independent of the activity life cycle that performs the long running task, and then reports back to the activity/view/fragment (while Mosby does offer the ability to tie the Presenter to the Activity lifecycle by cancelling the long running task and then restoring it, I really don't like that solution). They also have a way to keep track of states with built in options for common states: loading, content, error.

I think both libraries are great, the Mosby blog post by Hannes (http://hannesdorfmann.com/android/mosby/) had examples with explanations which made it easy to follow. 

###mosby






