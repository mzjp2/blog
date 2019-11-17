---
title: "Building a URL forwarder and scheduler with Go :link:"
layout: post
date: 2019-11-17 01:00
headerImage: false
tag:
- url
- Go
- http
- API

category: blog
author: zainpatel
description: "I built a HTTP webserver to forward to a new user-submitted link everyday using Go."
---

I've been eyeing [Go](https://golang.org) on and off for a while now and have been looking for an excuse to build something with it. So when I was tasked with building a URL forwarded and scheduler, I stopped myself from reaching for Python and ran `touch main.go` instead. Here's what I learnt...

## The tech stack

It's hosted at [qb.zainp.com](http://qb.zainp.com), with the DNS serving handled by Netlify, running on a spin-up-on-demand Heroku dyno, powered by Go's inbuilt `net/http` package and connected to a Heroku free-tier postgres instance (still living the poor student life :sweat_smile:).

## What is it anyway?

At work, we've been experimenting with post-it art. The latest idea was to create a QR code that we could change to link to something new every month. We quickly realised that manually changing the QR code, consisting of 900-something post-its every month wouldn't be fun, to say the least. Not to mention that a new link monthly would get old quickly. So I came up with the idea to let the QR code link to something dynamic and then control the forwarding there, which is what this project is. This also has the added bonus of changing the links at whatever interval we want.

So people can `POST` or `PUT` urls to [qb.zainp.com/link](http://qb.zainp.com/link) and the Go service automatically schedules the submitted URL for the next available day. A `GET` request to the same endpoint (aka someone opening it by scanning a QR code) will be auto-forwarded to the link of the day.

## How did using Go go?

It was great. I picked up the basics incredibly quickly, the tooling around it is great and the development experience is seamless, running `go build` or `go run` was too easy. The stark change of typing, coming from Python and the number of mistakes that caught during compile time, rather than runtime was mind-boggling.

I really like the concept around interfaces and structs, rather than heavy classes and objects in Python. The private/public distinction and having to design each piece of the service with regard to the interface you expose, even if you know the only person that'll be using it is yourself forced me to make good architecture decisions and structured my code in a nice way, which I'm all for. This is stark contrast with Python, where we assume everybody is a consenting adult and there is no real distinction between public and private methods or functions and so building interfaces just doesn't feel as nice.

I didn't make use of Go's concurrency pattern through this. At least, not directly. I believe that the `net/http` package handles every incoming request in a seperate goroutine, which I think is super cool, but anyway... I didn't use goroutines or channels, which is definitely a big part of what makes Go great, so I'm still looking forward to learning more about those, when to use them and implement them, either in this project or another.

## Not using a production web server

This in particular, blew my mind. With Python, when building an API, you use something like [Flask](https://flask.palletsprojects.com/en/1.1.x/) or [Django](https://www.djangoproject.com). Flask is my poison of choice, and it comes with a development web server for you to use within development. This seems fairly reasonable, on the face of it. At least, it certainly did to me. It does mean that when you want to run your webserver in production you need to get yourself a proper web server, probably something like nginX or Apache.


With Go, the default HTTP webserver that ships with Go's built-in `net/http` package *is* production-grade. In fact, I believe [dl.google.com](https://dl.google.com) is just `net/http`. In hindsight, that makes perfect sense. If you're going to ship a http server, you might as well (in true Go developer fashion) make it super performance, secure and production ready.

Admittedly, the service I've built isn't *truly* production ready, I need to build in some timeouts and configs available in `http.Server` to make sure that it's resilient against the mean mean people of the internet, but other than that, it also makes going from dev to prod really easy amd makes writing Docker containers for both environments really easy.


## Using postgres

I've used postgres before, both at Satavia and QB, but always relied on pre-existing architecture of the database. I've never really had to think through designing database schemas before, in fact, I've never really thought I could. The database I built here is pretty simplistic, just one table so far, but I'm planning to expand that. Postgres is nicer to use than I thought, even with the added complication of playing around with it through a Docker container. Not too much else to say on this point other than I hope I get the chance to build more complex stuff here.

As an aside, in general, I love the flexibility that comes with Go, in terms of interfaces and how plug-n-play, they are. If you want to build something that you cna write to, you just need to make it satisfy the `io.Writer` interface and chances are, pretty much every package that exists will be able to work nicely with it. This shines through in how database packages work in Go, wherein you import the general SQL package and then import a `lib/pq` postgres driver than pretty much builds out functionality for the interfaces defined in the `sql` package.

This means that your database work is pretty database agnostic (although that's not strictly true) and you need only learn one set of syntax.

## Where to go from here?

I want to build out this project a little more:

- add in the ability to view metrics from an API call
- allow viewing of metrics on a per-URL basis and a total basis.
- Build out a more complex database.
- Improve the underlying HTTP implementation to use timeouts and be resilient against resource attacks
- Learn more about Go, especially the concurrency patterns and use them
- Just learn more about how Go work internally, on a more theoretical level. I've heard that goroutines multiplex themselves on the same thread, so they're super-light and performant, but I don't even know how to begin unpacking that sentence.

## POST IT ART

pictures-to-come-soon.
