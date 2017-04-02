---
title: "CFNCluster"
teaching: 0
exercises: 25
questions:
- "What are the moving parts of cfncluster?"
objectives:
- "Take a Fourier transform or other non-trivial task as a basis for running a cluster compute task"
keypoints:
- "Each cloud has its approach to cluster computing; this is a common example."
---
## Prerequisites
- Log on to AWS


## In diagrams


Cfncluster uses three machine types: A Launcher, a Master and one or more Workers. The Launcher can as
easily be your laptop but for the sake of 'everything in the cloud' we will set it up as a small EC2 
instance. The Master is just what it sounds like, where the intelligence resides. The cloud elasticity
comes in how the Master spins up, tasks, and spins down the various Workers. Each of these Workers must
have custom software pre-installed so that we do not waste time on that process. 


Let's start with the classic *How It Works* diagram provided by AWS.


![cfn cluster workflow](/cloud101_cfncluster/fig/cfncluster_workflow.png)


The guts of this thing are the four sub-boxes; so all we need to do is understand how they function 
together, starting with a cronjob that runs once every minute on your Master instance.  A cronjob is 
a task that executes periodically on a Linux machine as part of the [cron](https://en.wikipedia.org/wiki/Cron) 
scheduler execution process.  In contrast Linux machines also run daemon processes which are always
active but generally quiet, waiting for some condition on the computer to trigger their active behavior.


You would think that a scheduler like **cron** all by itself could do everything
we show here; and like most ideas in UNIX it is (probably) possible but with a lot of manual effort. 
So **cfncluster** exists and works with a separate scheduler called **SGE** to accomplish our aims
with less effort.


The **cronjob** of interest is **publish_pending_jobs**, expanded here: 


![cfn cluster publish pending jobs](/cloud101_cfncluster/fig/cfncluster_publish_pending_jobs.png)


Notice that this queries something called a **queue manager** so let's define that next. We first
expand on the scheduler we mentioned above: **Sun Grid Engine** or **SGE**.
(A circa-2009 *for dummies* tutorial can be found 
[here](https://blogs.oracle.com/templedf/entry/sun_grid_engine_for_dummies).)


The SGE Master node runs a qmaster daemon. The SGE Worker nodes run an execution daemon. 
The qmaster daemon waits for you the *User* to submit a job. This job then goes through
a three state sequence: *pending* (sitting on the job queue), *running*, and then *done*. 
So we see that the **queue manager** is actually the SGE qmaster daemon. 

How do you submit a job? In our example we will use the **qsub** command (which is short for
*queue submit* of course): 

```
qsub fourier 53 19
``` 

## Deploy an EC2 Launcher

