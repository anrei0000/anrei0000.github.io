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
I create a programmatic access user. I download and store the credentials in one of my repos. 
* Name: `bitbucket`

_OnTheWay:_ Because I'm uploading secrets to git, I remember the [vault](https://www.hashicorp.com/products/vault/secrets-management) project, 
start to download it then add it to my [_next actions_](#more-reading) list.

## Create a role
* Type of trusted entity: `AWS Service`
* Service that will use this role: `EC2 service`
* Permissions: `AWSCodeDeployFullAccess`, `AmazonS3FullAccess`
* Name: `AWSCodeDeployRole`

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

_OnTheWay:_ Booting up my staging machine I bump into not being able to connect to it because of an 
`Offending ECDSA key in /c/Users/ciucu/.ssh/known_hosts:14`. 
Since I haven't assigned an elastic IP this is expected. I remove the key from known hosts then reconnect. 

_OnTheWay:_ In the mean time I am also updating to the latest `docker desktop version 2.1.0` and I can feel the anxious over what might break.
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
CodeDeploy > Getting Started > Create application

* Application name: `books-library`
* Compute platform: `EC2/On-premises`
* Deployment group: `books-deployment`
* Deployment group name: `DG1`
* Service role (we created this [here](#create-a-role)): `AWSCodeDeployRole`
* Deployment type: `In-place`
* Environment configuration: `Amazon EC2 instances`

This is the tag - value combination for the machine I am deploying to: `Name` - `Staging`

Check to make sure you get as many matches as you need (I get `1 unique matched instance.`).

* Deployment settings - deployment configuration: `CodeDeployDefault.AllAtOnce` 
* Load balancer: for the lulz, `Enable load balancing` even if there's nothing I can select in  `Choose a target group`
Aaand what do you know, I can't `Create deployment group` with this on. I'll just turn it off.
* Advanced: leave everything as is

Whoop whoop I get green.

## Setup bitbucket environment variables
After some digging around I had to find out what these are [here](https://confluence.atlassian.com/bitbucket/variables-in-pipelines-794502608.html),
so to sum up: click account image > bitbucket settings > account variables.
And I set the following key - value pairs:

* `AWS_ACCESS_KEY_ID` - `***`
* `AWS_SECRET_ACCESS_KEY` - `***`
* `APPLICATION_NAME` - `books-library`
* `AWS_DEFAULT_REGION` - `us-west-2`
* `DEPLOYMENT_CONFIG` - `CodeDeployDefault.AllAtOnce`
* `DEPLOYMENT_GROUP_NAME` - `DG1`
* `S3_BUCKET` - `codedeploy-hiring-books`

## Build the pipeline
The main steps should be the following
* Deploy to S3
* Tell CodeDeploy a new revision is ready
* Wait for CodeDeploy to perform deployment

We are doing this starting from [this python script](https://bitbucket.org/awslabs/aws-codedeploy-bitbucket-pipelines-python/src/73b7c31b0a72a038ea0a9b46e457392c45ce76da/codedeploy_deploy.py?at=master&fileviewer=file-view-default),
so I check it out locally `git clone git@bitbucket.org:awslabs/aws-codedeploy-bitbucket-pipelines-python.git`.

From this repo I am copying files over to my `hiring_books` repo and tweaking them when I have to.

* `appspec.yml`
* `codedeploy_deploy.py`
* `bitbucket-pipelines.yml`
* `scripts/install_dependencies`
* `scripts/start_server`
* `scripts/stop_server`

`#rolsuphissleeves` 'n gets to work.

I decided first I'll use a dummy `index.php` file which looks like this 
```
<?php echo "Hello World!";
```

As well, I just copy the default configurations and push to master. Yay! I get the first failed build!

Some issues with this first build:
* remote `pip` version is pretty old

I choose to ignore this for now since it's actually the version installed in the `python:3.5.1` container I'm running.
* the deployment folder must exist and it has to be empty

I create the emoty folder on my remote server.
* it didn't deploy the app

I start debugging.

I update my `bitbucket-pipelines.yml` file so that there are 5 steps instead of just 1. Didn't really help. Well... let's try and build the entire thing locally first so we can debug easily.

## Deploy locally
* Create a docker container
So I'm looking this up [here](https://confluence.atlassian.com/bitbucket/debug-your-pipelines-locally-with-docker-838273569.html) and [here](https://community.atlassian.com/t5/Bitbucket-questions/How-can-I-debug-my-Bitbucket-Pipelines-build-locally/qaq-p/136594) and I'm testing out the working command for opening a docker container replicating what should be run by CodeDeploy. Just make sure to run this in PowerShell, not in Git Bash, since that seems to fuck up some things..

```
docker run -it --rm --volume='//c//repo//books:/var/www/books' --workdir="//var/www/books" --memory=1024m --memory-swap=1024m python:3.5.1 /bin/bash
```

And I manage to open the container. In here I run `python codedeploy_deploy.local.py`.

Next error I'm currently debugging `botocore.exceptions.NoCredentialsError: Unable to locate credentials`. Found this [issue](https://github.com/spulec/moto/issues/1941) and from it changed the running command to 

```
AWS_ACCESS_KEY_ID=dummy-access-key AWS_SECRET_ACCESS_KEY=dummy-access-key-secret AWS_DEFAULT_REGION=us-east-1 python codedeploy_deploy.local.py
```

This produces `An error occurred (InvalidAccessKeyId) when calling the PutObject operation: The AWS Access Key Id you provided does not exist in our records.` which makes me think the previous error is environment related and that I should pass the correct records for testing.

Doing so gives a new error 

```
An error occurred (ApplicationDoesNotExistException) when calling the CreateDeployment operation: Applications not found for 363374259631
```

This made me think there's an issue with the IAM user/role. So I edited my `codedeploy_deploy.local.py` file and run it in the local container 

```
response = client.list_applications()
        print (response)
```

and got the response:
```
{'applications': [], 'ResponseMetadata': {'RequestId': '1cd2b159-9737-4d39-b618-a902a1effcae', 'HTTPHeaders': {'content-type': 'application/x-amz-json-1.1', 'date': 'Sat, 3 Aug 2019 22:48:33 GMT', 'content-length': '19', 'x-amzn-requestid': '1cd2b159-9737-4d39-b618-a902a1effcae'}, 'HTTPStatusCode': 200, 'RetryAttempts': 0}}
```



# More reading
* _Next actions_:
[The great CEO within](https://docs.google.com/document/d/1ZJZbv4J6FZ8Dnb0JuMhJxTnwl-dwqx5xl0s65DE3wO8/edit#heading=h.zdta3gwko2l) is a wonderful resource. You may read more about this in Chapter 3: Getting things done.