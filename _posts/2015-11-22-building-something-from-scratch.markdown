---
layout: post
title:  "Building something from scratch"
date:   2015-11-22 00:00:00
author: Gioia Ballin
categories: life
tags:	blogging
cover:  "/assets/worker.jpg"
---

I was working as a backend Python developer for about two years. One of my ongoing projects, at that time, regarded the refactoring of a big portion of unversioned code. Well, refactoring is not the correct word to say. This code base couldn't be simply taken and put into our repository for some simple makeover. It had to be designed and developed from scratch, again. Why? There were plenty of reasons in favour of the remaking:

- the code was using library dependencies which we wouldn't have maintained;
- the code wasn't integrated with our main [Twisted](https://twistedmatrix.com){:target="_blank"} application;
- the concurrency wasn't handled well, and it happened we found tons of processes referring to the entry-point module within our servers;
- the code was badly written (inconsistent naming, useless complex structure);
- the code hadn't been ever reviewed and, actually, we weren't sure it was accomplishing the objectives at 100.

So, I took the control of the situation, and I proposed and decided to make everything Twisted-based. There would have been a self-standing Twisted application doing all the stuff accomplished by the legacy code (hopefully doing them better) and an integration part, in which some of features would have been included into our main Twisted application server. That was the first time I had the chance of building an entire real-world application from scratch. It often happens to work on code that has been written by others, that has been already designed and for which the objectives are clear. So, the direction is clear. The fact is that you realise that the complexity considerably increases when you have to deal with the design and the implementation of coding solutions without having any existing reference of the direction to take. Soon, I began struggling with "Twisting" the application and, in retrospect, it was unavoidable. But, **each difficulty in programming can be turned into knowledge**. While you're struggling trying to understand new concepts, you're actually learning. Learning what? You're learning to solve real world problems. It's likely that someone else, in the world, will meet the same problems you're facing with right now. This thought drove me thinking about creating a blog. However, I also had lots of doubts.

> Would I have found the time to build a blog from scratch?
>
> Would I have been able to build a blog from scratch? Have I got enough skills?
>
> Would I have found the time to make a schedule for posting?
>
> Would I have stuck to my schedule?
>
> What would I have written?

At that time, I wasn't confident enough and, as consequence, I wasn't able to unravel my doubts. I temporarily set aside the idea about a blog, because I felt I needed a change in my life. A life change, a job change and even (maybe) a technology change. I moved to Milan and I began working for one of the most promising Italian startups. I didn't simply move from "somewhere really near Venice" to Milan "the metropolitan city", I moved also from the Python world to the Java world. Here is where the (mis)adventures started, in the Java world. In few months, I learnt a number of new things I couldn't even imagine before. I built things from scratch with an "almost unknown" technology. Like a sponge, I tried to absorb the knowledge from my colleagues. And with this experience, came the knowledge. Then again, the idea about a blog came out, along with one of the most important answers.

> What would I have written?
>
> Coding solutions to problems for which I struggled, results of my researches and, whatever I would have found interesting about coding and technology.

What about the other questions? I began browsing about blogging and coding. I read, I read and, again, I read. One night (yes, all this ludic activity is done at night) I was reading a [Jeff Atwood's blog post](http://blog.codinghorror.com/our-brave-new-world-of-4k-displays/){:target="_blank"} when I saw this comic coming from [dilbert.com](http://dilbert.com/){:target="_blank"}.

![](http://assets.amuniversal.com/31186d106ccb01301d50001dd8b71c47)
*The perfect representation of my existential doubts*

Even though I definitely prefer different blog posts by Jeff Atwood (discussions about displays doesn't attract me so much), that comic was inspirational. I couldn't stay reading others without trying doing something mine. Even if among "the others" there's Coding Horror, of which I'm a big fan.

What happened then? I tried to materialise the idea about a blog till this point, in which I'm writing my very first blog post. And, I don't know anything about the future. Does anyone know? For sure, I will try to [keep jabbing, keep shipping and keep firing](http://blog.codinghorror.com/how-to-achieve-ultimate-blog-success-in-one-easy-step/){:target="_blank"}.
