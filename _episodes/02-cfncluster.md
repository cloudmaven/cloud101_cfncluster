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
