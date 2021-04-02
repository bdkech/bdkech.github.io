---
layout: post
title:  "Playing with different frameworks"
date:   2021-04-02 10:20:59 -0500
---

* Table of Contents
    * Objective
        * Background and a walk down memory lane
        * The plan
    * The trials
        * Pachyderm
        * Kubeflow
    * Thoughts
{:toc}

# Objective
## Background and a walk down memory lane
So a little about how I work...I'd say about 70-75 percent of the time I work from the command line.  This is really an artifact of my time in graduate school.  Prior to that, I was a biologist that did programming and computer science courses.  I did seriously punish my iBook G4 through those years (I may do another blog post briefly recalling having to triage the hard drive with an iPod).  Starting my masters for Bioinformatics, I had a need for a new machine: Apple was no longer supporting the iBook line and OSX was coming around the corner.  With no where near enough money for a new Apple machine, so I had to settle for an eBook (For the youngin's out there, these were chromebooks before chromebooks...it was revolutionary at the time that it didn't have a CD Drive, you can google what those were).

Anyway, these simple machine forced me into learning *nix environment, thankfully.  I had to keep a Windows partition for Word and Powerpoint, but I had another for Ubuntu and was doing almost all of my coding server-side.  This is where I became mainly a vim user (all of our university VMs had it installed) and got into a workflow of remote working.  Tmux became a regular part of my toolkit, eventually managing my Python environments with virtual env, mosh instead of ssh to preserve connections, regular code versioning with SVN and Git.  Today I continue those habits but I've started adding in containerization and some light CI/CD. And I really enjoying managing and deploying my cloud resources with terraform so they're uniform and consistent.

This is an overly-long ancedote to emphasie I have an appreciation for working on remote machines and having behaviors that make replicating efforts seemless and efficient. I've used Makefiles and [Luigi](https://github.com/spotify/luigi) in the past to manage my Python and analytics workflows.  But with the rise of containerization I've wanted to make use of some frameworks like [Argo](https://argoproj.github.io/), [Pachyderm](https://www.pachyderm.com/), and [KubeFlow](https://www.kubeflow.org/) to make even more reproducible, redistributable code.

## The plan
So, really my plan was simple:  Download each of these frameworks and try to deploy them on a minikube, single node instance for tinkering.

I have a home server at my disposal, which is an HP workstation with 24 cores and about 32 gigs of RAM.  I've got minikube available so lets try deploying these kubernetes-based frameworks and recording my thoughts and experiences...

# The trials
I'll start with Pachyderm, then Kubeflow, and finally Argo.  The idea is to get each framework deployed, and do a minor test execution.  Most cases each have a getting started example and I think that'll do the trick for now.


## Pachyderm
Deploying Pachyderm was surprisingly straight forward, started up minikube, installed the pachctl tool, and did a local deployment.  After a few minutes I was up and running and doing the tutorial session.  Its a little neat the imagined workflow: develop some code, make a container, register some data, and write a specification to process the data.  The demo I worked through had an example with code wrapped into the container to do edge detection.

I could imagine getting into a routine of locally developing code, pushing to repo, and have a series of actions in place to rebuild and publish a container.  I like how any updates to the registered datasets kicks off a new run of the pipeline, but I'd like to test doing a round of model training and get an understanding of how scaling works, or if I can parametrize things in the specification.

One thing that was a bummer was that the Web UI is Enterprise-Only.  If you supply your email address when you do the local deployment you do get a 14 day trial.  With my limited time, currently, I'm not sure how much of the provenance and interdependencies you can check with the pachctl tool.  There is an inspect command so maybe there is a way to get that information out of the CLI, it wasn't immediately apparent and was starting to get beyond what I wanted to initially test.


## Kubeflow
So I had this at the top of my notes for KubeFlow:

>Man kubeflow, why can’t I quit you.  Lots of promise and bells and whistles, lots of headaches getting started.

I have to admit I've tried setting up KubeFlow before.  This has been the easiest deployment, but it still wasn't without hitches.  To be fair, KubeFlow does have a ton of added features and capabilities, so added complexity is acceptable.  To a degree.  One positive about all of this complexity is that I get to re-familiarize myself with Kubernetes again (getting pod states and logs, restarting deployments and resources).

First off I had to tear down minikube, it needs a number of extra parameters at startup compared to Pachyderm.  Afterwards I followed the directions for a single node setup with minikube on Linux on the KubeFlow website.  As I expected, a few pods went into CrashLoopBackOff.

![Not entirely unexpected](/assets/04-02-post/crashloop_kf.png)

In this case something wasn't write with the cache-related resources under the cache-deployer-deployment.  I suspected that if these weren't working some of the UI components wouldn't either, along with metadata tracking. Looking at the deployment logs indicated something wasn't right at all with the networking side, and most likely there was an issue with the istio deployment.

![Theres your problem](/assets/04-02-post/kubeflowevents.png)

So I looked through some of the issues on github and found this issue [Istio-Proxy fails on Kubeflow 1.2.0](https://github.com/kubeflow/kubeflow/issues/5447), which then led to this one:[How to start minikube with TokenRequest API enabled?](https://github.com/kubernetes/minikube/issues/7855#issuecomment-628287907).

In short, one the certificate files on the minikube start had to be changed.

Change

`minikube start --cpus 6 --memory 12288 --disk-size=120g --extra-config=apiserver.service-account-issuer=api --extra-config=apiserver.service-account-signing-key-file=/var/lib/minikube/certs/apiserver.key --extra-config=apiserver.service-account-api-audiences=api`

To

`minikube start --cpus 6 --memory 12288 --disk-size=120g --extra-config=apiserver.service-account-issuer=api --extra-config=apiserver.service-account-signing-key-file=/var/lib/minikube/certs/sa.key --extra-config=apiserver.service-account-api-audiences=api`

This got further, mysql still failed, along with some associated pods. Turns out I had to disable app armor per [mysql example failed: mysqld: Can't read dir of '/etc/mysql/conf.d/'](https://github.com/kubernetes/minikube/issues/7906).  Since this is a development and testing deployment I wasn't too worried about the change. Once you've done that you have to restart the mysql deployment.

`kubectl rollout restart deployment mysql -n kubeflow`

This does start to make me wonder about the experience about going into a "production" state with KubeFlow.  So this got me further along.  I'm still having a metadata writer in a crash loop, there is another [issue](https://github.com/kubeflow/kubeflow/issues/5458). I think I'm close enough that I can actually begin testing some components and get an idea of the experience.  With what I've got running I need to do some port forwarding to my remote host (its a headless machine) so I can access the UI.

`kubectl port-forward --address 0.0.0.0 -n istio-system svc/istio-ingressgateway 8889:80`

Seems to be up and running, actual testing pipelines will happen some other day.  I chose to start a notebook server, since it seems easy enough.  Namespace management could become a nightmare and that makes me wonder what the optimal team size would be for a single KubeFlow deployment, also I'm getting the feeling I'd be killing a fly with a cannon ball with KubeFlow at the moment.

Okay after taking awhile to pull the image (I only had a low CPU/Mem setting since this is on Minikube) the container has gone to a CrashLoopBackOff.  Checking the logs the container was hitting a “Permission denied” error because the user in the container ‘jovyan’ and [another issue](https://github.com/kubeflow/kubeflow/issues/1241).

I think I get the idea of how this is working…I’m not going to hunt down this particular issue, some of this is probably because of Minikube deployments, not having persistent volumes, etc.

Getting KubeFlow up and running took more time than I had anticipated.  It seems with each version there is a slightly different tweak I always have to do after deployments.  Argo will have to be on the examining table another time.

![GLORY](/assets/04-02-post/success.png)

# Thoughts

Okay, KubeFlow has *a lot* of bells and whistles.  Really the getting started section is only about installation.  I’d imagine at first I’d predominantly be using notebooks/pipelining.  KFServing, Fairing look really interesting for model serving and remote execution.  One aspect I need to shore up is data management patterns in the KF environment.  I’m also curious how a multi-tenant environment works, especially with the need for multiple namespaces.  Is there some way to do cross-namespace access?  If I make something in one space can I import/export from another?

Another issue is components are in various states of development (alpha/beta) and others report lagging documentation versions.  This has been an ongoing issue even in TensorFlow.  There are several points where installation or execution is relying on outdated commands and you have to dig into GitHub issues to find resolutions or appropriate version numbers.

It is not fair to compare KubeFlow to Pachyderm and Argo.  I think they’re much more focused solutions.  Its much like comparing Luigi to Airflow.  They (Pachyderm and Argo) feel a bit more appropriate for say, small teams or individuals working to share and iterate on code.  I could see advocating for KF in a start-up environment where the company was solely focused on ML/AI dev work.  I don’t think I’d enjoy managing the infrastructure *and* being a user of it.

I tried doing an Argo deployment some time ago, and I think I should have really paired it with my Pachyderm test first, then spent more time on KubeFlow.  But, the next session will be Argo.  Afterwards I’ll noodle on what I want to do : build some terraform infrastructure or perhaps take some of my hobby code and set it up in these frameworks?  Maybe even containers and GitHub are enough for me to be flexible with whatever workflow frameworks are out there.

Fun aside:  In grade school I loved the word "pachyderm."  Whenever we'd do exercises I'd identify elephants as pachyderms and ovals as ellipses much to the frustration of my teachers.  I guess I was going to be a scientist from a young age.
