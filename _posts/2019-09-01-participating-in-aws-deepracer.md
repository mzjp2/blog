---
title: "Participating in AWS DeepRacer :car:"
layout: post
date: 2019-09-01 01:00
headerImage: false
tag:
- machine-learning
- reinforcement-learning
- data-science
- AWS

category: blog
author: zainpatel
description: "I took part in the August virtual races in the AWS DeepRacer virtual league in my 3-person team, formed of QB colleagues. This post details what I've learnt from the experience."
---

So, what is AWS DeepRacer? It's Amazon's monthly competition where you train a reinforcement learning model and run it on that months track, ranking yourself against everybody else in the world.

## Update 

Turns out we placed 168th (out of roughly 1,300 teams) and came first in our internal company competition, bagging ourselves some goodies.

We also went to the AWS offices for a meetup centred around Deepracer, and as part of that, had the chance to try our model (where up to this point, had been tested, evaluated and deployed only on virtual cars) on a physical car on the physical track. 

This brought up all sorts of problems and issues that the basic simmulated AWS virtual environment just couldn't cpature, like the glare on the track from the overhead lights and how that translates to the camera feedback and what not. Our model didn't perform anywhere near as well as we'd expected on the physical track, as compared to the virtual one.

It was a great meetup and we had the chance to meet some of the people behind the entire league as well as some of the other people at the top of the leaderboard and learn some tips and tricks that we're excited to try in next months competition.

![](https://imgur.com/UTbjTIg.png)

## The track

August's track was the [Shanghai Sudu](https://aws.amazon.com/deepracer/schedule-and-standings/?p=drl&exp=btn&loc=4) track, arguably the hardest due to its two horshoe turns at either end, which wreak havoc on RL models.

![](https://deepracer-managed-resources-us-east-1.s3.amazonaws.com/track-resources/chinaalt_track.svg)

## Background

I went into this knowing nothing about reinforcement learning, at least, other than knowing it was about training an agent via a series of rewards and punishment for its actions, so I wasn't expecting much, and was hoping to learn and get hands on with it. I was part of a 3 person team, the other two being fellow QuantumBlack software engineers.

We worked on this every Friday afternoon, looking over our code together, suggesting changes and different approeaches, each of us taking ownership of one particular approach to try during the week and reporting back with resuults the next Friday.

I learnt loads and greatly enjoyed the experience, and look forward to continuing it next month for the Curulo Carera track.

## How does it work?

Right, so how does it actually work? You need to provide a reward function that takes in a whole [bunch of parameters](https://docs.aws.amazon.com/deepracer/latest/developerguide/deepracer-reward-function-input.html), like the position of the car, it's speed, heading, how far away it is from the centre, the two closest waypoints, width of the track, etc... and based off that information, return a reward, a float value.

You then select an action space, so how many different actions the car can take, where more actions usually allow for more flexibility and smoothness but also mean that training is slower and more computationally heavy. The typical action space we'd use was something like having possible headings of -30, -15, 15 and 30 degrees and speeds of 7 and 3.5. This created 8 different possible action combinations. 

You can then choose to tune the hyperparameters for the training, which I won't touch on too much, as I still don't have a very clear grasp of what each one means and largely used the default values.

You then submit the model for training, where it tries to navigate the track, getting rewards for its actions and slowly start to learn how to choose the right action to maximise the reward.

That means that we need to be careful with how we decide what rewards to give, because our agent (the car), we need to design a function that acts as a good proxy for "finish this track without falling off the track and going suitably fast!", whilst being careful of unintended side effects. If we tell it that moving away from the centre line is **really, really** bad, then it'll try and stick to it as much as possible, which means that on turns, it'll have to slow down and hug the centre line, ruinings its speed. Or if we tell it to go *very* fast, it'll just zoom off the track everytime because in the short amount of time that it goes very fast, we assign it a higher reward than it going moderate speed and not falling off the track.

## Our initial approaches

The first approaches we tried were fairly simplistic, such as rewarding the car when it was on or close to the centre line, with very simple decisionining and discrete rewards. This worked okay on the vanilla re:Invent 2018 track, not winning any prizes, but it'd be able to complete the track after an hour or so of training, but failed on the tricker Shanghai Sudu track, thanks to the horshoe turns where the car just couldn't navigate.

We also tried using a waypoint approach, where we'd take the closest previous and closest upcoming waypoint (handily provided to us by AWS), measure its heading and return a reward that was defined as a continuous function of the distance between our current heading and the "ideal" waypoint heading, where a higher reward was given for having a smaller distance, where the distance was calculated by a simple subtraction of the two headings in degrees.

This, whilst being an improvement over the "stay on the centre" approach on the re:Invent 2018 vanilla track, didn't work on the Shanghai Sudu track, for the same reasons. The waypoints are discretely positioned on the track and on horshoe turns do no nececssarily provide a good "ideal heading", meaning that our car would often overcorrect and fly off the track on these turns.

In fact, both approaches would make our car zig zag around the centre line quite often, the first because if the car was too much on the right, we'd punish it by rewarding it less, so it'd swerve to the left, and we'd reward it as it gets to the centre, but it doesn't react fast enough, so it overcorrects and ends up too far left, so we lessen its reward and it swerves to the right, ad nauseam. The second because the waypoints would change discrete as we travelled, so the ideal heading at any given point wasn't varying continuously like it should have, but was jumping between quantized states, giving rise to this rocky, jerky and 'swervy' motion to the car.

## What we realised

We eventually came to the conclusion that if we wanted a smooth motion and to overcome the horshoe turns, we'd need a reward function that keeps state. This was initially something we thought was impossible, because Amazon's input asks you to input a reward function and all it does is call this reward function with new parameters over and over again, after each action. So how do you keep state and use the history of the car to  determine the reward?

We worked around this by adding a class *outside* the reward function (something I haven't seen anybody talk about) which kept track of the previous state, including things like the previous timestamp, previous position, etc... and so allowing us to use the history (albeit, one timeframe in the past) of the car to help determine the reward.

## PID comes to our rescue

We did some research and came across the notion of a PID controller, which looked like a good candidate to solve our problem with swerving and rang some bells with regard to the problem it solved, where the "D" in "PID" would help cancel out the swerving from just using "P", which is essentially what our very first model did. Remember? It was using the position of the car. Wait, that isn't quite right. It was using the distance of the car from the centre, but that's just a proxy for the position of the car, so it would lead to this overcorrecting, jerky behaviour.

If we were able to use "derivative" to help curb this jerky movement, we'd be well on our way. What do we need to calculuate the derivative? Well, we need the difference in position over the difference in time. But that requires having the previous state... Aha!

We did have that, so our external state-keeping class was able to calculate the derivative and we could use that to curb the reward continuously and prevent the car from over correcting. 

This was the image that really drove it home:

![](https://upload.wikimedia.org/wikipedia/commons/a/a3/PID_varyingP.jpg)

where you can think of the blue line as being the centre line, using just position to calculate the reward being the purple line, where the car swerves back and foorth across the centre line in these large jagged spikes, using P+D being the red line, where the car is able to curb the approach, but gets stuck to one side, and the green line being P+I+D (where I is for integration and helps curb long term biases from using D). It vaguely reminds me of the proof of the inclusion-exclusion principle. 

This was our first approach that actually compeleted the track. We ended up with a time of roughly 18s and a rank of roughly 220 (out of 1400).

Implementing I as well, which meant using our class to keep the state of the overall long term bias and correct for it when it became large enough gave our model a large boost, and we ended up with a time of 13.3s, and a rank of 173 at the end of the month, with the top time being arund 8 seconds.

## What next?

We've got a few ideas for how to improve our model for next months competition, which I won't get into now.

All in all, I've learnt a lot about reinforcement learning and had a lot of fun building and tweaking the model, and of course, racing it. 

I'd like to thank my two colleagues for their help and participation, I wouldn't have been able to do it without them and it wouldn't have been half as much fun without being able to do it together, nor would I have learnt anywhere near as much.
