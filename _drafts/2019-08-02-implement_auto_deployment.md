---
layout: post
title: "Auto-deployment from Bitbucket to AWS"
date: 2019-08-02
---

Here is my work in progress at doing CI. 
I have a brand new project called `hiring_books` which deserves its own auto install to my cloud.

What I have is the following
* AWS services
* bitbucket hosted repo
* google.com

To kick things off I quickly check out options offered by bitbucket and I notice AWS CodeDeploy which sounds like something I would want.
I also find [this tutorial](https://hackernoon.com/deploy-to-ec2-with-aws-codedeploy-from-bitbucket-pipelines-4f403e96d50c).

I'm taking the usual route of following the tutorial until my work is done or I need a new tutorial.

## Create AWS user
I create the programmatic access user `bitbucket`. I download and store the credentials in one of my repos. 
* Name: 
Because I'm uploading secrets to git, I remember the [vault](https://www.hashicorp.com/products/vault/secrets-management) project, 
start to download it then add it to my [_next actions_](#more-reading) list.

## Create a role
* Type of trusted entity: AWS Service
* Service that will use this role: EC2 service
* Permissions: AWSCodeDeployFullAccess, AmazonS3FullAccess
* Name: AWSCodeDeployRole

In the listing click the role name, then go to the trust relationship tab and update the trust policy.
Make sure the region matches what you deploy to.

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": [
            "ec2.amazonaws.com",
            "codedeploy.us-west-2.amazonaws.com"
        ]
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

## Create an S3 bucket
* Name: `codedeploy-hiring-books`

For the rest, just leave the settings as they are.

## Update my EC2 instance

I already have 2 EC2 instances set up (one for staging, one for prod). Only thing left to do is check the IAM role attached to it.
I realize that I already had a role defined for my instance, so now I just update my existing role with what is described in [Create a role](#create-a-role). 

Booting up my staging machine I bump into not being able to connect to it because of an 
`Offending ECDSA key in /c/Users/ciucu/.ssh/known_hosts:14`. 
Since I haven't assigned an elastic IP this is expected. I remove the key from known hosts then reconnect. 

In the mean time I am also updating to the latest `docker desktop version 2.1.0` and I can feel the anxious over what might break.
... Whew, all is well! Onwards.

## Install CodeDeploy on Ubuntu 16.04
Following the [guide here](https://docs.aws.amazon.com/codedeploy/latest/userguide/codedeploy-agent-operations-install-ubuntu.html) looks like this
```bash
sudo apt-get install ruby
sudo apt-get install wget
cd /home/ubuntu
wget https://aws-codedeploy-us-west-2.s3.us-west-2.amazonaws.com/latest/install
chmod +x ./install
sudo ./install auto
```

The install script lets me know I actually need `ruby` version `2.0`. 
So I do `apt-get install ruby2.0` then rerun `sudo ./install auto`.

After the above steps I check the install
`sudo service codedeploy-agent status`
and I get `The AWS CodeDeploy agent is running as PID 10465` which is good.

## Setup AWS CodeDeploy
CodeDeploy - Getting Started - Create application

* Application name: `books-library`
* Compute platform: `EC2/On-premises`


# More reading
* _Next actions_:
[The great CEO within](https://docs.google.com/document/d/1ZJZbv4J6FZ8Dnb0JuMhJxTnwl-dwqx5xl0s65DE3wO8/edit#heading=h.zdta3gwko2l) is a wonderful resource. You may read more about this in Chapter 3: Getting things done.