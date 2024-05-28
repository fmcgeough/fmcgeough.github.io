---
layout: post
title: Your Data Says Your Code Is Ugly
date: 2017-03-15 08:53:13
description: What does your data representation say about your code?
categories: code
tags: code
---

As someone who has some expertise in relational databases, I'm called upon
to dig into problems where code is misbehaving because some code
is running into some relational db data that is unexpected. Like
much in life, the folks on the development side are quick to
fault the database. This parallels the usual response of typical DBAs - which
is to quickly fault the developers. This situation doesn't get much better with
folks who claim they are "full-stack developers". Someone declaring that they
are a "full-stack" developer is best met with a sigh at this point. Its a
marketing term more than a statement of fact.

In one long session where I was cleaning up data so that an app would function
properly I thought about how much you could imply about the code based on what
I was doing to get the app to function. Here's some general thoughts :

The issue that I was cleaning up in this case was :
**Data associated with deleted users or accounts left hanging around in database**.

I see this problem extremely frequently. Here's my thoughts when I see this situation.

First, from a relational database perspective I may be looking at a system where
the designers "got creative". That is, they didn't use normal relational tables where
you could simply use a foreign key to delete associated data when a delete happens.
Or perhaps the people who built the system were not even aware that such a facility
exists. If that's the case then this is probably just the
start of problems that I'll be called upon to clean up for this customer. Its fairly
easy to identify this situation. Is there some random character field in the database
that actually holds an integer value that is a primary key to some table? OK, then
this system is going to be nothing but trouble. Its fairly amazing how often I see this
given all the problems it creates.

Lets talk about deleted data. From a business perspective you may want to save off data
even if the user or account (or service or whatever) is no longer around and operational.
Fine. You should not keep it in the main path of your operational system. Its just an
invitation for bloat and confusion. Save deleted data off to some archival or reporting
system if you must. But don't leave it hanging out in your operational path.

Assuming you don't actually have a need to store this data and its just being left
around inadvertently then what can we assume about the code? Well, we can probably safely
assume that there was no clear specification about what to do when the object was
deleted. And that there is no test to ensure that the data is removed. That seems like
a safe assumption.

The fact that no test was developed for this situation is probably indicative that most
non-sunny day scenarios are not tested either. There may be lots of holes in this software
related to when things go temporarily bad. Like network interruptions, for example.

Further, if some of the data is deleted but some is left behind then
we may have software that is being evolved by folks who weren't part
of the original development team. That the original code was convoluted or obtuse enough
that it was difficult for new developers to see that some additional work needed to
be done when new objects were connected to the existing ones.

Further, given that problems are arising because data was erroneously left behind
we can probably safely assume that managers within the company are now more nervous
about any changes to the software because it appears that the developers don't actually
appear to know what they are doing. This will tend to yield even worse problems because
of a reluctance to address some of the core issues in the software that may require
rethinking and simplifying the design - which could mean a large code change.

This may be carrying on a bit far for what may be a far simpler situation but they are
some of things that cross my mind when I'm called in to help. After I've cleaned up the
data issue, I then want to examine the code to try and figure out how bad the situation
is and how likely it is to continue to cause the original (or worse) problems.

But, in general I think the cleanliness of your state information (stored in a relational
db or in a no-sql database) is a good indication of the general fitness and trustworthiness
of the code that you've written. If you can't be trusted with your state information then
what exactly can you be trusted with?
