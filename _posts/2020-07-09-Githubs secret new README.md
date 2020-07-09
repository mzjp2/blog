---
title: "Github's secret new user README's :octocat:"
layout: post
date: 2020-07-09 22:00
headerImage: false
tag:
- github
- feature

category: blog
author: zainpatel
description: "Github covertly launched README's for user profiles. Here's how to get your own and inspire yourself from others."
---

From what I gather, it looks like Github covertly launched a new feature that allows users to create a `README.md` that renders on their user profile page, which is a pretty big shakeup of the github profile page, in my opinion. I heard about it through one of my friends, who heard about it from a friend, who heard about it on Twitter.

This is what it looks like (from [my current profile](https://github.com/mzjp2)):

![mzjp2 github profile screenshot](https://imgur.com/I06ZD1l.png)

To get your own, you need to create a new repository in your account with the same name as your account, so in my case, with my account name being [mzjp2](https://github.com/mzjp2) I needed to create a _public_ repository called `mzjp2`, accessible at [github.com/mzjp2/mzjp2](https://github.com/mzjp2/mzjp2) with, at least, a `README.md`.

![new repo creation screen](https://imgur.com/meBGUIz.png)

You can add additional files to your repository (I have a folder of `svg` icons located in there that I use as images in my `README`), but the only thing that ultimately gets rendered is the `README.md`. It follows the same rendering rules and follows the same Github markdown spec as all the other `README`'s you're used to in any other repo.

One caveat that I ran into is that you shouldn't use relative links in the `README`. For example, I had an `<img src="icons/linkedin.svg">` reference in my `README.md` which rendered fine locally since my repository structure had `icons/` and `README.md` on the same level _and_ rendered fine on the `README` view inside the `mzjp2/mzjp2` repo, but did _not_ render well on my Github user profile. I switched to using aboslute links (pointing at Github's user content static URL) and everything worked fine after that.

As you can imagine, people have gone wild with this new feature, building everything from a guestbook to a communal chess game on their profile. Some of the ones I've seen so far:

* [github.com/WaylonWalker](https://github.com/WaylonWalker) that uses Netlify's `_redirect` feature to always point to his latest blog post on his README and also incorporated the clever `visitor` badge that I shamelessly stole.
* [github.com/timburgan](https://github.com/timburgan) a Github staff member (I wonder how much time he had to play around with this :thinking:) that built a community chess tournament using buttons (to move your chess piece) that opens a Github issue prepopulated text, which triggers a Github action that updates the `README.md` file with the new state of the game. :clap:
* [github.com/saadeghi](https://github.com/saadeghi) who has the infamous Chrome dino game as a GIF playing. Minimalism, anyone?

and plenty more. If there are any interesting ones you've come across, feel free to ping me an email and I'd love to check it out!