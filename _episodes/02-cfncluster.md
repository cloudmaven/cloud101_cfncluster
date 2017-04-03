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
- Refer: [Cloudmaven](http://cloudmaven.org) and its [EC2 page](http://cloudmaven.org/aws_ec2.html)
- You have a properly sanitized AWS account
- Your IAM User credential file (public and private ID strings) is in a secure location (*never* on like GitHub!)


## In diagrams


Cfncluster uses three machine types: A Launcher, a Master and one or more Workers. (You can also think
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
[here](http://blogs.oracle.com/templedf/entry/sun_grid_engine_for_dummies).)


The SGE Master node runs a qmaster daemon. The SGE Worker nodes run an execution daemon. 
The qmaster daemon waits for you the *User* to submit a job. This job then goes through
a three state sequence: *pending* (sitting on the job queue), *running*, and then *done*. 
So we see that the **queue manager** is actually the SGE qmaster daemon. 


How do you submit a job? In our example we will use the **qsub** command (which is short for
*queue submit* of course): 


```
qsub fourier 53 19
``` 


### Auto Scaling Group and Nodewatcher Diagrams


Clusters deployed within CfnCluster use the configure file to determine:


- initial_queue_size: Initial size of the ComputeFleet Auto Scaling Group (ASG)
- max_queue_size: Maximum size of the ComputeFleet ASG.


Two CloudWatch alarms are created. They monitor a custom Amazon CloudWatch metric that is published 
by the Master node of the cluster. This metric is called **pending**. These CloudWatch alarms call 
ScaleUp policies associated with the ComputeFleet ASG. This is what handles the automatic addition 
of compute nodes when there are pending tasks in the cluster: Nodes are added until the alarms stop
or the max_queue_size is reached.  Auto Scaling also has a Cloudwatch alarm to remove instances that
are no longer needed; hence there is a corresponding ScaleDown policy. 


Each Worker instance in the ComputeFleet ASG runs a process called **nodewatcher**.  This process 
monitors the instance and if idle AND close to the end of the current hour the Worker is removed
from the ComputeFleet ASG using an API call.  Again the overall concept here is that the cluster
scales itself back as long as it has adequate capacity for the current perceived work load and
as long as it is at or above its configuration minimum. To this end there is a third boolean
parameter **maintain_initial_queue_size** that can be set to true in the config file. 


Here are the corresponding diagrams: 


![cfncluster auto scaling](/cloud101_cfncluster/fig/cfncluster_autoscaling.png)


Notice that Simple Queue Service (SQS) is an important part of this flowchart. It is 
discussed further below as the fourth of our four daemon processes in CfnCluster. 
Before turning to SQS though let us dispense with the Nodewatcher as follows.

![cfncluster node watcher](/cloud101_cfncluster/fig/cfncluster_nodewatcher.png)


This is a straightforward IF AND statement that causes Workers to be terminated.


### Simple Queue Service (SQS)

SQS is used in cfncluster as the online/offline messaging system.  The sqswatcher daemon
running on the CfnCluster Master monitors for SQS messages emitted produced by Auto Scaling: 
When an instance comes online it submits "instance ready" to SQS which is picked up by 
sqs_watcher running on the Master. These messages are used to notify the queue manager 
(in our case the SGE qmaster daemon) that new instances are online. The reverse is also
true: When instances are terminated the SQS is used to notify the Mater that they are
no longer available.


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
5. Once a Master is available: Log in and configure it to execute some task 
6. Use a shell script to fill up the job queue with tasks
7. Let SGE "notice" new instances and give them jobs as they become available


- Auto Scale Group (ASG)
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

- Refer to the [EC2 page here](http://cloudmaven.org/aws_ec2.html)
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
a web page like [this one](http://cfncluster.readthedocs.io/en/latest/getting_started.html).

```
cfncluster create PIT0
```

The cluster you create includes a head node. By default this will be a small EC2 instance (T2). 
On the AWS console in your browser you can monitor your progress 



### Our job/Worker program

This C code performs part of a Fourier transform on a simple dataset. The idea would be to bring 
these results together. To simulate harder work there is a built in 3.5 minute 'sleep' at the 
beginning of the program. 

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

#define MAX_N 32768
void main(int argc, char **argv)
{
    sleep(210); // sleep for 210 seconds (since the subsequent computation will be nearly instantaneous)
    if (argc < 3) { printf("\nfourier N i: i'th coeff of an N-element signal\n\n\n"); exit(0); }
    int N = atoi(argv[1]); 
    int i = atoi(argv[2]);          printf("\nSignal length %d, term %d.\n\n", N, i);
    if (N < 0 || N > MAX_N || i < 0 || i >= N) { exit(0); }
    double s[MAX_N], ar = 0.0, ai = 0.0, pi = acos(-1.0), dN = (double)N, di = (double)i;
    double dShft = (double)(N/2), dScl = (2.0*pi*5.0)/dShft, dGScl = 2.0/dShft;
    for (int n = 0; n < N; n++) {             // signal generator block
        double dn = (double)n;                  // convert index to a floating point value
	double x = (dn - dShft) * dScl;         // ... to a number on [-5 * 2pi, 5 * 2pi]
	double xg = (dn - dShft) * dGScl;       // ... also to a number on [-2, 2]
	double g = exp(-xg*xg);                 // ... and get the Gaussian of the latter
	double m = sin(x);                      // ... and the sine of the former
	s[n] = g*m;                             // ... and compile their product into the signal vector s[]
	// printf ("%d,%lf\n"n, s[n]);
    }
    for (int n = 0; n < N; n++) {             // FT block
        double dn = (double)n;                //   dn is the sum index
        double exp_arg = -2.0*pi*(dn/dN)*di;  //   argument of the exponential
	double real_n = cos(exp_arg);         //   real component of the exponential
	double imag_n = sin(exp_arg);         //   imag component of the exponential
        ar += s[n]*real_n;                    //   accumulate
	ai += s[n]*imag_n;                    //      "
    }
    printf ("\n\ncoefficient %d = (%lf, %lf).\n\n\n"), i, ar, ai); exit(0);
}
```

Compile this program as follows: 

```
gcc -o fourier fourier.c -lm 
```

Now we better save this image of the Master so that our Workers will know how to run fourier without compiling it. 


Here is a script that submits multiple fourier jobs to the job queue:

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



## More legacy content

We are logged in to the cfncluster Master node.  Notice that 'qhost' produces an empty listing; 
just the 'global' process (which kilroy thinks is the Master)


```
% qhost
HOSTNAME                          ARCH            NCPU NSOC NCOR NTHR   LOAD   MEMTOT  etcetera
------------------------------------------------------------------------------------------------
global                            -                  -    -    -    -      -        -
```

Create a shellscript file 'kilroy.sh':


```
#!/bin/bash
sleep 4
echo "kilroy shout out from $(hostname)"
```

Make this script executable and submit it to the queue


```
% chmod a+x kilroy.sh
% qsub kilroy.sh
```

As machines spin up they appear in **qhost** output on the Master node. As jobs exist they
appear as the output of **qstat** on the Master node. 


```
% qhost
HOSTNAME                          ARCH            NCPU NSOC NCOR NTHR   LOAD   MEMTOT  etcetera
------------------------------------------------------------------------------------------------
global                            -                  -    -    -    -      -        -  etcetera
ip-172-31-17-24                   1x-amd64           1    1    1    1   0.51   995.6M  etcetera

% qstat
job-ID  prior   name       user        state  submit/start at          queue 
----------------------------------------------------------------------------
     1  0.55500 kilroy.sh  ec2-user    qw     02/11/2016 22:24:35
```


These tasks run pretty quickly (minutes). When the script runs to completion qstat will again show 
an empty queue.  The Worker node will stay available for the balance of the hour that I have rented 
it for; since I already paid for it. Then it evaporates if it is not doing anything.  


As these tasks run: Where is the output going? We have a shared filesystem and the Workers are writing to the
home directory on the Master node. 


```
% ls
kilroy.sh kilroy.sh.e1 kilroy.sh.e2 kilroy.sh.o1 kilroy.sh.o2
```


These ".o1" and ".e1" files are respectively standard output and standard error.


How does configuration work? It is split between the Launcher and the head node. 


This would be a good place to put a kilroy insert on movable targets: ip address changing.


kilroy fragment: 


[CfnCluster config documentation](http://cfncluster.readthedocs.org/en/latest/configuration.html)


The directory on Launcher is .cfncluster with two files: *config* and *cfncluster-cli.log*. The 
latter is the log file produced by prior cluster activity. 


To see the prior configuration run 'configure':  


```
% cfncluster configure 
```


## Second time around: Running a job


- ssh to Master
- Create the fourier.c program, compile it
- Use qsub or qsh to submit several tasks


Our outputs will indicate their respective inputs...


Here is a .cfncluster/config file with an Elastic Block Storage (**EBS**) element of interest.
Recall this file sits on the Launcher instance.


```
[aws]
aws_region_name = us-west-2

[cluster default]
vpc_settings = public
key_name = MPIPlay
ebs_settings = helloebs
shared_dir = /hello
ec2_iam_role = hello_world
initial_queue_size = 5
max_queue_size = 5

[vpc public]
master_subnet_id = subnet-37513940
vpc_id = vpc-a60a4ec3

[global]
update_check = true
sanity_check = true
cluster_template = default

[ebs helloebs]
ebs_snapshot_id = snap-f67581a4
```

Here is a kilroy problem: A fragment qsub script file with some cool purpose that Kilroy lost track of...

```
#!/bin/sh
#$ -cwd
#$ -N helloworld
#$ -pe mpi 5
#$ -j y
date

/bin/rm hostfile

while read line; do
    echo $line | awk '{print $1}' >> hostfile
done < $PE_HOSTFILE

/usr/lib64/openmpi/bin/mpirun -hostfile hostfile ./hw.x > helloall.out
```

