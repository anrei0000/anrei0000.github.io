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
sudo apt-get install ruby2.0
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

## (just in case) Uninstall CodeDeploy
```
sudo apt-get remove codedeploy-agent
sudo apt-get purge codedeploy-agent
sudo rm -rf /opt/codedeploy-agent /var/log/aws/codedeploy-agent
``` 
I did this because it seemed there was some weird caching going on.. or at least old files being used somehow.

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

## Debug locally

* Create a docker container

So I'm looking this up [here](https://confluence.atlassian.com/bitbucket/debug-your-pipelines-locally-with-docker-838273569.html) and [here](https://community.atlassian.com/t5/Bitbucket-questions/How-can-I-debug-my-Bitbucket-Pipelines-build-locally/qaq-p/136594) and I'm testing out the working command for opening a docker container replicating what should be run by CodeDeploy. Just make sure to run this in PowerShell, not in Git Bash, since that seems to fuck up some things..

```
docker run -it --rm --volume='//c//repo//books:/var/www/books' --workdir="//var/www/books" --memory=1024m --memory-swap=1024m python:3.5.1 /bin/bash
```

And I manage to open the container. In here I run `python codedeploy_deploy.local.py`.

* Error 1

Next error I'm currently debugging `botocore.exceptions.NoCredentialsError: Unable to locate credentials`. Found this [issue](https://github.com/spulec/moto/issues/1941) and from it changed the running command to 

```
AWS_ACCESS_KEY_ID=dummy-access-key AWS_SECRET_ACCESS_KEY=dummy-access-key-secret AWS_DEFAULT_REGION=us-west-2 python codedeploy_deploy.local.py
```

* Error 2

Running the above produces `An error occurred (InvalidAccessKeyId) when calling the PutObject operation: The AWS Access Key Id you provided does not exist in our records.` which makes me think the previous error is environment related and that I should pass the correct records for testing.

* Error 3

I pass the correct credentials but now get

```
An error occurred (ApplicationDoesNotExistException) when calling the CreateDeployment operation: Applications not found for 363374259631
```

This made me think there's an issue with the IAM user/role. So I edited my `codedeploy_deploy.local.py` file like this 

```
response = client.list_applications()
        print (response)
```

* Error 4

...and got the response:
```
{'applications': [], 'ResponseMetadata': {'RequestId': '1cd2b159-9737-4d39-b618-a902a1effcae', 'HTTPHeaders': {'content-type': 'application/x-amz-json-1.1', 'date': 'Sat, 3 Aug 2019 22:48:33 GMT', 'content-length': '19', 'x-amzn-requestid': '1cd2b159-9737-4d39-b618-a902a1effcae'}, 'HTTPStatusCode': 200, 'RetryAttempts': 0}}
```

 I also add a print statement `print(deploymentResponse)`. This shows me the real error message I've been hunting for. 
```
The IAM role arn:aws:iam::363374259631:role/AWSCodeDeployRole does not give you permission to perform operations in the following AWS service: AmazonEC2. Contact your AWS administrator if you need help. If you are an AWS administrator, you can grant permissions to your users or groups by creating IAM policies.
```

I added `AmazonEC2FullAccess` permission to the role and retried.

* Error 6

This produces the error 
```
The overall deployment failed because too many individual instances failed deployment, too few healthy instances are available for deployment, or some instances in your deployment group are experiencing problems.
```

To debug this I ssh onto the EC2 machine where the codedeploy agent is running and do
```
tail -f /var/log/aws/codedeploy-agent/codedeploy-agent.log
```

* Error 7

This is now showing me the error
```
The specified key does not exist.
```

Apparently I was passing the wrong bucket identifier and I had also removed the `str`
function from the variables in `codedeploy_deploy.local.py`.

* _ByTheWay_

These is how I reset the code deploy agent on the machine.
```
sudo service codedeploy-agent stop
sudo service codedeploy-agent start
```

* Error 8

This next one took a while for my brain to stop fuzzing.
I was also tailing the log in my staging instance but no ideas. The error was
```
put_host_command_complete(command_status:"Failed",diagnostics:{format:"JSON",payload:"{\"error_code\":5,\"script_name\":\"\",\"message\":\"Script at specified location: scripts/install_dependencies run as user root failed with exit code 126\",\"log\":\"\"}"}
```

Also the same root cause but different error 

```
Script at specified location: scripts/install_dependencies run as user root failed with exit code 126  
```
What was going on is that I was changing configurations but not re-archiving the files.

* _Finally_

Somehow I realised only now that to test locally I need to first start the machine and install prerequisites.
```
docker run -it --rm --volume='//c//repo//books:/var/www/books' --workdir="//var/www/books" --memory=1024m --memory-swap=1024m python:3.5.1 /bin/bash

apt-key adv --keyserver keyserver.ubuntu.com --recv-keys AA8E81B4331F7F50 # for some reason this is needed locally
apt-get update
apt-get install -y zip vim # vim used locally for debug
pip install boto3==1.3.0
```

But then, to run a deploy I also need to rerun the `zip`. I was actually running this only randomnly and this made the debug process harder since errors were somehow inconsistent.
```
zip -r /tmp/artifact.zip *
AWS_ACCESS_KEY_ID=*** AWS_SECRET_ACCESS_KEY=*** APPLICATION_NAME=books-library AWS_DEFAULT_REGION=us-west-2 DEPLOYMENT_CONFIG=CodeDeployDefault.AllAtOnce  DEPLOYMENT_GROUP_NAME=DG1 S3_BUCKET=codedeploy-hiring-books python codedeploy_deploy.py
```

Funnily enough, I found the [guide here](https://medium.com/@team_62166/how-to-deploy-laravel-on-amazon-web-services-a-detailed-step-by-step-tutorial-2aff3f180a1f) 
which helped with issues in the config files:

* Files were not getting where I wanted them to go.

This below is how it should be obviously !?

```
files:
  - source: /
    destination: /var/www/books
```

... of course initially it wasn't obvious, since I was originally passing `source: /index.html`

## Deploy to staging
The code is now getting to the machine and I'm working on getting the env right:

Since this is turning out to be such a long thing, I feel that I didn't give enough warning that I'm not doing a tutorial but more of a log of my processes as I go through the code. At this point, I started the post 6 days ago. 

Lots of issues appear since I'm not just using new stuff but reusing (like my staging machine). This is what actually happens IRL. You don't always get to `do-release-update`, instead you get to

* [Start to update php](https://www.rosehosting.com/blog/install-php-7-1-with-nginx-on-an-ubuntu-16-04-vps/)

```
sudo add-apt-repository ppa:ondrej/nginx-mainline
sudo apt-get update -y
sudo apt-get install php7.2 php7.2-curl php7.2-dev php7.2-gd php7.2-mbstring php7.2-zip php7.2-mysql php7.2-xml php7.2-opcache php7.2-redis php7.2-fpm php7.2-imap php7.2-mcrypt
sudo apt-get install php7.1 php7.1-common php7.1-cli
sudo apt-get install systemd
...
```

Take a moment to reconsider life choices ... decide to `do-release-upgrade` up to ubuntu `16.04` then

* [Install `php7.2`](https://thishosting.rocks/install-php-on-ubuntu/)




# More reading
* _Next actions_:
[The great CEO within](https://docs.google.com/document/d/1ZJZbv4J6FZ8Dnb0JuMhJxTnwl-dwqx5xl0s65DE3wO8/edit#heading=h.zdta3gwko2l) is a wonderful resource. You may read more about this in Chapter 3: Getting things done.
* Github makes CI/CD free for public repos