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
scheduler execution. You would think that a scheduler like **cron** all by itself could do everything
we show here; and like most ideas in UNIX it is (probably) possible but with a lot of manual effort. 
So **cfncluster** exists and works with a separate scheduler called **SGE** to accomplish our aims
with less effort.


The **cronjob** of interest is **publish_pending_jobs**, expanded here: 

![cfn cluster publish pending jobs](/cloud101_cfncluster/fig/cfncluster_publish_pending_jobs.png)

Notice that this queries something called a **queue manager** so let's define that next. 


## Deploy an EC2 Launcher

1. Go to EC2

```
conda create -n env_name python
```
![](/cloud101_webframework/fig/02-elasticbeanstalk-0001.png)

3. You are now ready to create your new project. From the File menu, select New Project then Django. Fill in the location where you want to save your project (Here I am saving it to /Users/Amanda/PycharmProjects/beanstalkdemo). You can your project whatever you want. Then choose your interepreter. Click the little button on the right of the Interpreter text box, select "Add local" then find the virtual environment that you created earlier. If you created your environment using conda, it should be in your home directory under anaconda. For example, my virtual environment will be stored in ~/anaconda/envs/aws-env/. You will want to add the Python interpreter created in that virtual environment. Here, I am using Python 3.6.0. Click create.

![](/cloud101_webframework/fig/02-elasticbeanstalk-0002.png)

4. You should now have a project with a folder structure that looks something like this:

![](/cloud101_webframework/fig/02-elasticbeanstalk-0003.png)

5. Create a folder named .ebextensions in the root of the project tree. This is the where AWS Beanstalk configuration files reside. Create a file named django.config inside the .ebextensions folder. Paste the following (remember to substitute beanstalkdemo for your project name):

~~~
option_settings:
  aws:elasticbeanstalk:container:python:
    WSGIPath: beanstalkdemo/wsgi.py
~~~

6. Next...

~~~
appdirs==1.4.3
Django==1.10
packaging==16.8
pyparsing==2.2.0
six==1.10.0
~~~

7. 
