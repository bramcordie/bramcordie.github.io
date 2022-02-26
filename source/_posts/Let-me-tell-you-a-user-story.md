---
title: Let me tell you a user story
date: 2022-02-26 22:20:32
tags: [agile, uses stories, rant]
---

There once was a developer frustrated with user stories telling him how to implement them… 

There is this anti-pattern I’ve been dealing with, where technical assumptions slip into the description of user stories. When using a ticketing system like JIRA, it’s easy for anyone to create an issue and slap the story type on it. Often with good intentions, suggestions slip in on how the technical implementation might work. This causes noise and distracts from the actual user story. 

When this happens, it becomes harder to figure out the real problem that I’m supposed to solve. Let’s compare two stories and their description: 
 
__An existing user can log in to the application.__

There is a user that has already registered and confirmed his email address by opening his Outlook client and clicking the link with a confirmation token defined in a query string. 
When he goes to the website and does not already have a cookie that gives him a session he gets to see the login page. 
He then clicks on the username label which puts his cursor in the accompanying field. There he types Karlor and presses tab. This puts focus on the password field, where he types acornTorso993. By pressing enter the login button, a HTTP POST request is sent, making the login button fade out to a more saturated color and becoming disabled until the login process fails or succeeds. The login does succeed through sheer willpower. Somewhere on the page that now shows up, there is a visual hint that Karlor is now in fact logged in, like the word DASHBOARD. 


__As an authorized user I see a list of my most recent projects after I authenticate, so I can start working on them again.__

That’s it, no fluff. 

In the second example it is immediately clear what the useful behavior is for the user. And even though the first example can be perfectly written out as a specification it is harder to see the real value. Let’s assume both stories end up in the backlog and won’t be worked on for some time. Imagine a future where authentication methods have evolved from being a simple username and password field to something more secure. The description of the first example makes a bunch of assumptions that will not age well when the world arounds it changes. It also stifles innovation when it comes to finding a suitable technical solution. 

Premature optimization is taken to a whole new level. Some of these stories already assume things are going to be slow and then try to shoehorn in a solution.  

> Hey, I know aggregating all this info will probably slow, so let’s use AJAX to fetch it in the background.  
 
This makes no sense when the goal of the story is having the user fill out any missing information before she can continue her work. She’s still supposed to wait until every detail is known, before she can take action. Fetching that info with extra AJAX requests will arguably make the process slower. 

The people implementing the story can also feel like they can’t be trusted to come up with their own suitable solution. The experience of having a meaningful discussion with other developers on how to tackle problems, in a pair-programming session or when preparing for the next sprint meeting for example, gets replaced with a weird puppet show. This can create the illusion of doing meaningful work but ultimately you’re just writing code instead of solving problems. 

![speech bubbles](/images/story-bubbles.png)

I’m not arguing that work should not be done upfront but I would rather see a meaningful context, for everyone involved. So, they can apply their own set of skills effectively.

If you have any thoughts or ideas about this blog post that you would like to share, head on over to it's GitHub discussion:
[https://github.com/bramcordie/bramcordie.github.io/discussions/2](https://github.com/bramcordie/bramcordie.github.io/discussions/2)
