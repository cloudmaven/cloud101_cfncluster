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
## Prerequisites, admonitions
- Log on to AWS
- Refer: [Cloudmaven](http://cloudmaven.org) and its [EC2 page](http://cloudmaven.org/aws_ec2.html).
- You have a properly sanitized AWS account
- Your IAM User credential file (public and private ID strings) is in a secure location (*never* on like GitHub!)


## In diagrams


Cfncluster uses three machine types: A Launcher, a Master and one or more Workers. (You can also worry
about shadow Masters that jump in if the Master fails; which in turn points up the notion of a robust
cluster that can carry on despite failures... which in turn leads to failure-driven strategies; cf
the AWS Spot market.)

The Launcher is as the name implies just about getting things set up. It can as easily be your laptop 
but for the sake of 'everything in the cloud' we will set it up as a small EC2 instance. The Master is 
just what it sounds like: where the cluster intelligence resides. The cloud elasticity comes in how 
the Master initiates, manages, and closes out the various *jobs* and *Workers*.  *jobs* are compute 
tasks that are managed by means of a queue and *Workers* are EC2 instances.  Each Worker should have 
custom software pre-installed (if it is needed for *jobs*) so that time is not wasted. We will keep an
eye on how this is done through the following process.


Let's start with the classic *How It Works* diagram provided by AWS. We will talk through the 
various components before doing the easy part of walking through an example.


![cfn cluster workflow](/cloud101_cfncluster/fig/cfncluster_workflow.png)


The guts of this thing are the four sub-boxes in *Cluster Management Process*; so if we 
understand how they function together we will be in good shape.  


We begin with a cronjob that runs once every minute on your Master instance.  A cronjob is 
a task that executes periodically on a Linux machine as part of the [cron](http://en.wikipedia.org/wiki/Cron) 
scheduler execution process.  We note that Linux machines also run daemon processes which are always
active but generally quiet/passive, waiting for some condition on the computer to trigger their active behavior.


You would think that a scheduler like **cron** all by itself could do everything we show here; and like 
most solution ideas in Linux it is possible but it would require considerable manual effort.  Rather 
we use **cfncluster** working with a different scheduler called **SGE** to take advantage of an existing
solution. This solution includes a lot of detail management that has arisen as cluster computing has
evolved. One *SGE* advocate says 'powerful, flexible, scalable, useful'. But note that other schedulers
are available: SLURM, Toruqe, OpenLava, ... so you have options.


Ok, so the **cronjob** of interest is **publish_pending_jobs**, expanded here: 


![cfn cluster publish pending jobs](/cloud101_cfncluster/fig/cfncluster_publish_pending_jobs.png)


Notice that this queries something called a **queue manager** so let's define that next. We first
expand on the scheduler we mentioned above: **Sun Grid Engine** or **SGE**.
(A circa-2009 *for dummies* tutorial can be found 

kilroy here kilroy https://blogs.oracle.com/templedf/entry/sun_grid_engine_for_dummies kilroy .)


The SGE Master node runs a qmaster daemon. The SGE Worker nodes run an execution daemon. 
The qmaster daemon waits for you the *User* to submit a job. This job then goes through
a three state sequence: *pending* (sitting on the job queue), *running*, and then *done*. 
So we see that the **queue manager** is actually the SGE qmaster daemon. 

How do you submit a job? In our example we will use the **qsub** command (which is short for
*queue submit* of course): 

```
qsub fourier 53 19
``` 






## Walkthrough

### Key vocabulary

- PIT: Project Identifier Tag, a unique string you designate like 'himat'
- queue: A stack of jobs managed by a scheduler (in our example SGE)
- Master: A VM that manages the job queue 
- Worker: A VM charged that executes jobs
- Scheduler: Software that manages/executes jobs on a queue 
  - Includes SLURM, Torque, OpenLava and SGE
- qsub: (Linux) submit a job to a processing queue
- qstat: (Linux) get queue status
- qdel: (Linux) delete a job from the queue
- host: List Worker nodes
- IAM User Keys: There are two: The AWS Access Key ID and the AWS Secret Access Key ID


### Strategy

These steps depend upon some pre-configuration, already done prior to the class. 


- We created a Virtual Private Cloud with a public subnet for this exercise. 
- We created an associated Internet Gateway and a Security Group. 
- The Security Group permits ssh in from *any* location on the internet
  - In passing this is not best practice because your work is visible from anywhere. 

In what follows you will want to have a Project Identifer Tag or PIT handy. This is just 
a short ID string that you will use to tag everything you create. Mine PIT = 'kilroy'.
Herein I use *PIT* or *kilroy* interchangeably.

Our strategy:

1. Start up a Launcher EC2 on AWS 
2. Log in to this machine as 'ec2-user' using ssh and update it (standard practice)
3. Install cfncluster and create a cluster on AWS '*kilroy*'
4. On the AWS console watch your progress on the CloudFormation service 
5. 

- Autoscale group
- Cloud Formation service
- Cloud Watch service
- Simple Notification Service
- EC2 instance: Launcher with *cfncluster* installed
- cfncluster creates a cluster that includes...
  - An EC2 Master instance
    - Recognizes Worker EC2 instances as they are *added* to the resource pool
    - Has some memory
    - Includes task software
    - Shell script to launch multiple jobs to the SGE processing queue


### Create an EC2 instance cfncluster Launcher

- Refer to the kilroy EC2 page here kilroy kilroy http://cloudmaven.org/aws_ec2.html kilroy
- It will be the 'base of operations' for the compute task
  - Give it a PIT name like 'rob101_Launcher'
  - It can be small and cheap to operate, for example a **T2.micro**.
- Choose the Amazon Linux AMI 
  - This has AWS tools already installed; but we will update and install cfncluster
- The default EBS volume is fine but could be expanded as needed per data 
- We will place everything in a single Region/AZ/etceter
    - Region = us-west-2 (Oregon)
    - AZ = Zone C
    - VPC = Virtual Private Cloud cloud101_vpc
    - Subnet = cloud101_public_subnet
    - Internet Gateway = ... kilroy
    - Route Table entries ... kilroy
    - Security Group = cloud101_securitygroup (ssh allowed from anywhere: 'hi risk')
- Review and launch
  - Download a new key pair
  - You can use an existing key pair as well; be sure you have it on hand
    - This will be a '.pem' file extension. 
      - If you are Windows using PuTTY you will need to convert to a .ppk file format
      - This is done using the PuTTYGen application
- Once it spins up (green dot, ip address present) ssh to this EC2 as ec2-user

Notice that the VPC and so on already exist: Preliminary spadework we are sparing you in this
course. 

### Log in and configure the Launcher

You should now be able to log in to your cfncluster Launcher using ssh (or PuTTY on Windows)
where your login name is 'ec2-user'. You do not enter a password as you are using your .pem
(or .ppk) file to authenticate. 

Once logged in you will update your machine, install the cfncluster tools, configure 
cfncluster and create a new named cluster. This in turn will lead to the last steps to 
run a large-scale compute task.

On your EC2 Launcher:

```
sudo yum update
sudo pip install cfncluster
sudo pip install --upgrade cfncluster
cfncluster configure
```

Running *configure* will produce a config file within the .cfncluster directory in your home
directory. (Use 'ls -al' to see that this exists.) A good way to get the configure steps 
correct is to follow the details at 
a web page like kilroy this one kilroy  kilroy http://cfncluster.readthedocs.io/en/latest/getting_started.html kilroy .

```
cfncluster create PIT0
```

The cluster you create includes a head node. By default this will be a small EC2 instance (T2). 
On the AWS console in your browser you can monitor your progress 



### Our job/Worker program

This C code performs part of a Fourier transform on a simple dataset. In ensemble 

```
// fourier.c performs a simple Fourier transform of a small data vector
//   This is intended as a fast but non-trivial compute task.
//     Signal = a Gaussian envelope x sine wave with about 10 cycles
//     FT: i is the i'th term of the FT; a_i is the coefficient, complex 
//     a_i = Sum over n running 0 to N-1 of x_n * CE
//     CE is a complex exponential e^{ -2 * pi * i * k * n / N }
#include <stdio.h>
#include <stdlib.h>
#include <math.h>
#include <unistd.h>

```

Compile this program: 

```
gcc -o fourier fourier.c -lm 
```

Now we better save this image of the Master so that our Workers will know how to run fourier without compiling it. 


Here is a script that submits fourier jobs to the queue using qsub a number of times:
```
qsub /home/ec2-user/fourier 137 0
qsub /home/ec2-user/fourier 137 1
qsub /home/ec2-user/fourier 137 2
qsub /home/ec2-user/fourier 137 3
qsub /home/ec2-user/fourier 137 4
qsub /home/ec2-user/fourier 137 5
qsub /home/ec2-user/fourier 137 6
qsub /home/ec2-user/fourier 137 7
qsub /home/ec2-user/fourier 137 8
qsub /home/ec2-user/fourier 137 9
qsub /home/ec2-user/fourier 137 10
qsub /home/ec2-user/fourier 137 11
qsub /home/ec2-user/fourier 137 12
qsub /home/ec2-user/fourier 137 13
```
  

{% include links.html %}


































## Deploy an EC2 Launcher

