# Introduction

A while back, I sat down with Thinkster founder Eric to talk about how we could take Thinkster’s courses to the next level. We’d talk about this topic a number of times and we kept coming back to the one thing we overwhelmingly agreed on: There is no clear path from toy projects to production-ready applications. We spent three hours drilling into this statement. Here are the three key takeaways from that conversation:

1. Everyone knows this is a problem.
2. No one is doing anything about it.
3. That sucks.

We parted ways that night with the goal of brainstorming some solutions and meeting up a few days later to compare notes.

By the end of the night, Eric was pitching me on the idea of a new set of courses whose goal it was to bridge this gap. They would go into more detail than past courses had and the end result would be an MVP-level app. We would teach Thinkster students to build some real and substantial. Something they would be proud of and something that would help them make an even larger leap into full-time employment as professional software developers.

Eric was kind enough to let me write the Django course. This course is our answer to the pleas of would-be Djangonauts. Our hope is that we have set a path for you to follow and that we have left nuggets of wisdom for you to internalize along the way. I can’t promise this will be easy — nothing worth having ever is — but, if you’re willing to put in the work, then we are confident this course will be an early checkpoint on your road to success.

## About the Author

Let me introduce myself.

My name is James Brewer. For the past few years, I’ve been working professionally as a software engineer in Silicon Valley where I’ve built enterprise web applications from scratch, worked with teams to build and improve consumer products that reach millions of users every month, and am currently building a software company of my own.

In addition to this tutorial, I’ve written for Thinkster previously with some amount of success. I was the first author to join outside of Thinkster’s founders and the courses I’ve written have helped over 100,000 readers become better Django developers.

I’ve been working with Django for longer than I care to admit and have worked on more projects than I can count. I’ve contributed to Django’s documentation and, through my previous work with Thinkster, have answered more questions about Django than I thought could be asked.

## Goals Of This Course

Our goal for in writing this course is to bridge the gap between toy projects and production-ready products. To accomplish this, we have designed an API that will power the an MVP-level product. This course will walk you through building that API yourself.

We do not concern ourselves with questions like “Will it scale?”. These questions are, for now, unimportant and only serve to slow you down. Instead, we’ve focused on crafting something that will get the job done while following a single principle: make good decisions. We actively avoid things we know to be bad and we frequently make trade-offs to balance getting it done and getting it done right. Done is better than perfect.

Note: If you’re interested in a more advanced course covering topics like scaling, deployment, etc, give us a shout in the [Thinkster Pro](#) Slack channel.

## Final Product

The API we are going to build together will support the following features:

### Authentication

What good is a web app that users can’t log into? We will handle registration and login in this chapter. We’re also going to replace Django’s default session-based authentication with token-based authentication using [JSON Web Tokens](https://jwt.io/). We will introduce the concept of authorization (or permissioning) and make sure that only the logged in user can edit their own information. 

This will be the longest chapter because there are so many concepts to be introduced.

### Profiles

After all the security stuff is handled, we will need a way to store information about each user that isn’t related to authentication. Things like a biography, the user’s avatar, etc.

To do this, we will introduce user profiles and start experimenting with Django’s relationships by creating a one-to-one relationship between the user and profile classes. This means every user will have exactly one `Profile` object that gets created automatically when they register their account.

### Articles

Our fancy new web app wouldn’t be much fun if all you could do was sign up and log in. To show that we’re fun people, we’re going to introduce the concept of an article. Articles can be posted by any user for other users to read, favorite, and comment on. Think of it like a blog where anyone can post new content.

### Comments

“Going viral” is all the buzz these days. To make our site viral-friendly, we’re going to create a way for users to comment on the articles we just built. It’ll be just like Reddit!

### Following

Continuing our discussion of creating relationships, users should be able to follow one another. We’re going to introduce the concept of unidirectional, many-to-many relationships. Think of these relationships in terms of Twitter vs. Facebook. On Facebook, if we’re friends, then that means I am your friend and you are my friend. Contrast this with Twitter. If you follow me, I could be following you, but I don’t have to be. We will take the Twitter approach because it makes more sense for our app.

### Favoriting

Being able to favorite Articles is good for two reasons. First off, it’s lets you stroke the author’s ego. Secondly, it’s like your own personal bookmarking system. When a user favorites an article, we will create a relationship between the two. Then, if that user is looking for an article they read a while back that they enjoyed, they can just look at their list of favorited articles instead of scrolling back through their feed.

### Tagging

One of the more difficult questions to answer when building a content platform like the one we’re building is “How do users find content that is interesting to them?” That’s a toughy because there are so many options. On one end of the spectrum you have machine learning which can figure out what each user is interested in and can recommend content based on that knowledge of the user. On the other end, you’ve got a simple tagging system that asks the users to do the heavy lifting by assigning tags to the articles they publish.

Since we’re just looking to build an MVP, we’re going to use the latter approach. We will leave the machine learning for after we raise our C round and can throw tens of millions of dollars at the problem!

### Pagination

As our site grows and becomes more popular, our library of content will grow enormously. Loading and rendering 10,000 articles with a single API request is a bad idea for performance reasons. To get around this, we’re going to have to break up those 10,000 articles into multiple smaller requests. This is where pagination comes into play.

Pagination lets us retrieve 20 articles at a time instead of retrieving all of the articles in the database at once. Django REST Framework gives us some tools to make this simple.

### Filtering

We will also support filtering articles on a few different dimensions: by author, by tag, and by the user who favorited the article. This will give us another opportunity to help users find articles that will interest them. Maybe you’re starting to see a trend here. The goal of the API is to support the front end in giving users a pleasurable experience.

## Django and Django REST Framework Primer

We do assume some prior experience with both Django and  Django REST Framework. We will be covering a lot of ground in this course and having to explain every new concept from the ground up would do more harm than good.

Django’s introductory tutorial covers the basics of Django and will give you sufficient understanding to complete this course.

{x: complete the django tutorial} 
Complete [The Django Tutorial](https://docs.djangoproject.com/en/1.9/intro/tutorial01/)

If you’re interested in diving deeper into what Django has to offer, the API reference and the topical guides are amazing resources. You should get familiar with both if you plan to use Django for a production-ready application.

{x: check out the django api reference} 
Check out the [Django API Reference](https://docs.djangoproject.com/en/1.9/ref/)

{x: check out the django topical guides} 
Check out the [Django Topical Guides](https://docs.djangoproject.com/en/1.9/topics/).

Django REST Framework also has it’s own tutorial that covers most of what the library has to offer.

{x: complete the drf tutorial} 
Complete the [Django REST Framework Tutorial](http://www.django-rest-framework.org/tutorial/1-serialization/).

## Let’s get started

That’s all you need to know for now. To get started, clone the [conduit-django](https://github.com/brwr/conduit-django) repository and follow the instructions in the file called `README.md` to get everything installed and running.

{x: lets do this} 
I’ve cloned the repository and I’m ready to go. Let’s do this.