---
layout: post
title:  "Getting ducks in a row"
date:   2021-04-24 09:55:59 -0500
---

* Table of Contents
    * Prepping for the next home project
        * The plan
    * The reality :)
    * The questions
{:toc}

# Prepping for the next project

## The plan

So, last post was about me tinkering with some MLOps frameworks.  I was wanting to test developing a pipeline a little outside the norm.  I've also been curious about implementing a Q-Learning algorithm.  Briefly, reinforcement learning take the concept that you train an agent, given a set of inputs (or states), to act in an environment to generate a new state and an associated award.  So, I thought to myself, why not train a model to play video games?

I admit, I enjoy video games, see previous posts.  Lately I've gotten a kick out of the speedrunning community. In this case, it isn't enough that you play video games, but you use various strategies, glitches, and tricks to complete games as fast as possible.  There's a subcategory called tool assisted speedruns (TAS).  I looooove these.  [In this case players are using scripts and emulators to do tricks that'd be humanly impossible, or to inject arbitrary code into games](https://youtu.be/mSFHKAvTGNk). Seriously check out that video, its insane.  Anyway, I was curious about trying to teach an AI to play by making the objective completion time.  I'd also thought about mining the LUA scripts that some people write for playing games as a starting seed.  Basically, I want to teach an AI to do crazy things in games...and maybe it'll actually finish a level.

Before I got too ahead of myself I figured it'd be best to try and do some basic work using [OpenAIs gym](https://gym.openai.com) and see if I can get a model playing something.

# The reality :)

Okay, so, setting this up was actually a headache.  I really got stalled out on a small bug that is a result of my setup.  Whoops.

See, to use the gyms you actually have to run the games for the agents to "play" in.  I'm running all of this on a remote unix workstation running a GPU for my training. I had to spend a bit of time making sure I had the right libraries set up for running gyms.  But there was another issue.  When they're running they need to open a window to render the game and the agent playing.  Normally this isn't too much of an issue, I'd use X-forwarding on ssh (ssh -Y you@yourhost). *Normally* this isn't an issue.  I'm doing devleopment on a M1 MacBook Air.  I spent a long time trying to figure out why I was getting a generic OpenGL rendering error in python when running a minimal example.  I thought it was some conflict between the NVidia drivers and OpenGL.  But nope, it was a quirk with X11Quartz on the mac.  I needed to open a console and add:

`defaults write org.xquartz.X11 enable_iglx -bool true`

Then I could actually render the remote emulator, locally.  The FPS's were low, I'm not sure if that is an artifact of the streaming or the agent playing the game and how it refreshes.  Next was figuring out how to train an agent to control games that were outside of the included library.  There was a fun talk from [PyCon 2018 from freeCodeCamp](https://youtu.be/NIG4BZ8VpF4) and it looks like some emulators now support controlling the game via LUA scripts and sockets.  So there'll be some extra work I need to do to get all of that set up and container-ized.  I was hoping to spend more time on doing that than chasing phantom driver bugs.  But c'est la vie.

The weather is getting nicer, so that means more home and lawn care (plus allergies).  So development may be slowed for a bit.  It can be a bit frustrating, but these are the tasks I enjoy, they teach a few lessons about implementations beyond yaMes (yet-another-MNIST-example): resource and platform engineering, workflow consideration, and objective management.

# The questions

* Does anyone know if the the OpenAI gyms or associated emualtors are to be run in a 'headless' mode?  I don't really want to commit resources to basically always have a laptop on to stream the agent playing.
* Is this even going to work in kubeflow?  My intuition is saying I may need a docker-compose style environment to properly set everything up (agent + running game environment + infrastructure)
