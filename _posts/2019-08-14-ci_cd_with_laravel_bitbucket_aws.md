---
layout: post
title: "CI/CD with Laravel, Bitbucket and AWS"
date: 2019-08-14
---

TL;DR I tell you the story of how I got [this demo Laravel project](https://www.books.lpgfmk.xyz) 
hosted on [this Bitbucket repo](https://bitbucket.org/anrei0000/hiring_books/src/master/) installed on an EC2 instance with a pipeline configured to build on pushes to master.

# Overview

* [Intro](#intro)
* [Create AWS user](#create-aws-user)
* [Create a role](#create-a-role)
* [Create an S3 bucket](#create-an-s3-bucket)
* [Update my EC2 instance](#update-my-ec2-instance)
* [Install CodeDeploy on Ubuntu 16.04](#install-codedeploy-on-ubuntu-1604)
* [(just in case) Uninstall CodeDeploy](#just-in-case-uninstall-codedeploy)
* [Setup AWS CodeDeploy](#setup-aws-codedeploy)
* [Setup bitbucket environment variables](#setup-bitbucket-environment-variables)
* [Build the pipeline](#build-the-pipeline)
* [Debug locally](#debug-locally) (Fun times here, `error` gets initialized)
* [Deploy](#deploy-to-staging)
* [Test deployment](#test-deployment)
* [Conclusions](#conclusions)
* [More reading](#more-reading)

# Intro
Disclaimer: Here is my work in progress at doing CI. **Please** be aware that if you follow this you will get errors similar to what I'm describing. You will _likely_ also get the fixes working. This is an "as I go" description, only lightly reviewed at the end, expect things to break at first, then expect solutions that worked for me.

That said, I have a brand new project called `hiring_books` which deserves its own auto install to my cloud.

What I have is the following
* AWS services
* bitbucket hosted repo
* google.com

To kick things off I quickly check out options offered by bitbucket and I notice AWS CodeDeploy which sounds like something I would want.
I also find [this tutorial](https://hackernoon.com/deploy-to-ec2-with-aws-codedeploy-from-bitbucket-pipelines-4f403e96d50c).

I'm taking the usual route of following the tutorial until my work is done or I need a new tutorial.

# Create AWS user 
I create a programmatic access user. I download and store the credentials in one of my repos. 
* Name: `bitbucket`

_OnTheWay:_ Because I'm uploading secrets to git, I remember the [vault](https://www.hashicorp.com/products/vault/secrets-management) project, 
start to download it then add it to my [_next actions_](#more-reading) list.

# Create a role
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

# Create an S3 bucket
* Name: `codedeploy-hiring-books`

For the rest, just leave the settings as they are.

# Update my EC2 instance

I already have 2 EC2 instances set up (one for staging, one for prod). Only thing left to do is check the IAM role attached to it.
I realize that I already had a role defined for my instance, so now I just update my existing role with what is described in [Create a role](#create-a-role). 

_OnTheWay:_ Booting up my staging machine I bump into not being able to connect to it because of an 
`Offending ECDSA key in /c/Users/ciucu/.ssh/known_hosts:14`. 
Since I haven't assigned an elastic IP this is expected. I remove the key from known hosts then reconnect. 

_OnTheWay:_ In the mean time I am also updating to the latest `docker desktop version 2.1.0` and I can feel the anxious over what might break.
... Whew, all is well! Onwards.

# Install CodeDeploy on Ubuntu 16.04
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

# (just in case) Uninstall CodeDeploy
```
sudo apt-get remove codedeploy-agent -y
sudo apt-get purge codedeploy-agent
sudo rm -rf /opt/codedeploy-agent /var/log/aws/codedeploy-agent
``` 
I did this (several times :)) ) because it seemed there was some weird caching going on.. or at least old files being used somehow.

# Setup AWS CodeDeploy
CodeDeploy > Getting Started > Create application

* Application name: `books-library`
* Compute platform: `EC2/On-premises`
* Deployment group: `books-deployment`
* Deployment group name: `DG1`
* [Service role](#create-a-role): `AWSCodeDeployRole`
* Deployment type: `In-place`
* Environment configuration: `Amazon EC2 instances`

This is the tag - value combination for the machine I am deploying to: `Name` - `Staging`

Check to make sure you get as many matches as you need (I get `1 unique matched instance.`).

* Deployment settings - deployment configuration: `CodeDeployDefault.AllAtOnce` 
* Load balancer: for the lulz, `Enable load balancing` even if there's nothing I can select in  `Choose a target group`
Aaand what do you know, I can't `Create deployment group` with this on. I'll just turn it off.
* Advanced: leave everything as is

Whoop whoop I get green.

# Setup bitbucket environment variables
After some digging around I had to find out [what these are](https://confluence.atlassian.com/bitbucket/variables-in-pipelines-794502608.html),
so to sum up how to find them: click account image > bitbucket settings > account variables.
And I set the following key - value pairs:

* `AWS_ACCESS_KEY_ID` - `***`
* `AWS_SECRET_ACCESS_KEY` - `***`
* `APPLICATION_NAME` - `books-library`
* `AWS_DEFAULT_REGION` - `us-west-2`
* `DEPLOYMENT_CONFIG` - `CodeDeployDefault.AllAtOnce`
* `DEPLOYMENT_GROUP_NAME` - `DG1`
* `S3_BUCKET` - `codedeploy-hiring-books`

# Build the pipeline
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

#rolsuphissleeves 'n gets to work.

I decided first I'll use a dummy `index.php` file which looks like this. (Later, I'm confused about this decision since it seems it was kinda forgotten...) 
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

# Debug locally

* Create a docker container

So I'm looking this up [here](https://confluence.atlassian.com/bitbucket/debug-your-pipelines-locally-with-docker-838273569.html) and [here](https://community.atlassian.com/t5/Bitbucket-questions/How-can-I-debug-my-Bitbucket-Pipelines-build-locally/qaq-p/136594) and I'm testing out the working command for opening a docker container replicating what should be run by CodeDeploy. Just make sure to run this in PowerShell, not in Git Bash, since that seems to fuck up some things..

```
docker run -it --rm --volume='//c//repo//books:/var/www/books' --workdir="//var/www/books" --memory=1024m --memory-swap=1024m python:3.5.1 /bin/bash
```

And I manage to open the container. In here I run `python codedeploy_deploy.local.py`.

* Error = 0

```
botocore.exceptions.NoCredentialsError: Unable to locate credentials
```

Found [this issue](https://github.com/spulec/moto/issues/1941) and from it changed the running command to 

```
AWS_ACCESS_KEY_ID=dummy-access-key AWS_SECRET_ACCESS_KEY=dummy-access-key-secret AWS_DEFAULT_REGION=us-west-2 python ./scripts/local_deploy.sh
```

* Error++

Running the above produces 
```
An error occurred (InvalidAccessKeyId) when calling the PutObject operation: The AWS Access Key Id you provided does not exist in our records.
``` 

which makes me think the previous error is environment related and that I should pass the correct records for testing.

* Error++

I pass the correct credentials but now get

```
An error occurred (ApplicationDoesNotExistException) when calling the CreateDeployment operation: Applications not found for 363374259631
```

This made me think there's an issue with the IAM user/role. So I edited my `codedeploy_deploy.local.py` file like this 

```
response = client.list_applications()
        print (response)
```

* Error++

...and got the response:
```
{'applications': [], 'ResponseMetadata': {'RequestId': '1cd2b159-9737-4d39-b618-a902a1effcae', 'HTTPHeaders': {'content-type': 'application/x-amz-json-1.1', 'date': 'Sat, 3 Aug 2019 22:48:33 GMT', 'content-length': '19', 'x-amzn-requestid': '1cd2b159-9737-4d39-b618-a902a1effcae'}, 'HTTPStatusCode': 200, 'RetryAttempts': 0}}
```

 I also add a print statement `print(deploymentResponse)`. This shows me the real error message I've been hunting for. 
```
The IAM role arn:aws:iam::363374259631:role/AWSCodeDeployRole does not give you permission to perform operations in the following AWS service: AmazonEC2. Contact your AWS administrator if you need help. If you are an AWS administrator, you can grant permissions to your users or groups by creating IAM policies.
```

I added `AmazonEC2FullAccess` permission to the role and retried.

* Error++

This produces the error 
```
The overall deployment failed because too many individual instances failed deployment, too few healthy instances are available for deployment, or some instances in your deployment group are experiencing problems.
```

To debug this I ssh onto the EC2 machine where the codedeploy agent is running and do
```
tail -f /var/log/aws/codedeploy-agent/codedeploy-agent.log
```

* Error++

This is now showing me the error
```
The specified key does not exist.
```

Apparently I was passing the wrong bucket identifier and I had also removed the `str`
function from the variables in `codedeploy_deploy.local.py`.

_By the way_, this is how I reset the code deploy agent on the machine.

```
sudo service codedeploy-agent stop && sudo service codedeploy-agent start
```

* Error++

This next one took a while for my brain to stop fuzzing.
I was also tailing the log in my staging instance but no ideas.
```
put_host_command_complete(command_status:"Failed",diagnostics:{format:"JSON",payload:"{\"error_code\":5,\"script_name\":\"\",\"message\":\"Script at specified location: scripts/install_dependencies run as user root failed with exit code 126\",\"log\":\"\"}"}
```

Also the same root cause but different error 

```
Script at specified location: scripts/install_dependencies run as user root failed with exit code 126  
```
What was going on is that I was changing configurations but not re-archiving the files when deploying.

* _Finally_

Somehow I realised only now that to test locally I need to first start the machine and install prerequisites.
```
docker run -it --rm --volume='//c//repo//books:/var/www/books' --workdir="//var/www/books" --memory=1024m --memory-swap=1024m python:3.5.1 /bin/bash

apt-key adv --keyserver keyserver.ubuntu.com --recv-keys AA8E81B4331F7F50 # for some reason this is needed locally
apt-get update
apt-get install -y zip vim # vim used locally for debug
pip install boto3==1.3.0
```

But then, to run a deploy I also need to rerun the `zip`. I was actually running this only sometimes and this made the debug process harder since errors were somehow inconsistent.
```
zip -r /tmp/artifact.zip *
AWS_ACCESS_KEY_ID=*** AWS_SECRET_ACCESS_KEY=*** APPLICATION_NAME=books-library AWS_DEFAULT_REGION=us-west-2 DEPLOYMENT_CONFIG=CodeDeployDefault.AllAtOnce  DEPLOYMENT_GROUP_NAME=DG1 S3_BUCKET=codedeploy-hiring-books python codedeploy_deploy.py
```

Funnily enough, I found the [guide here](https://medium.com/@team_62166/how-to-deploy-laravel-on-amazon-web-services-a-detailed-step-by-step-tutorial-2aff3f180a1f) 
which helped with the next problem.

* Error++

Files are not getting where I wanted them to go.

This below is how it should be obviously !?

```
files:
  - source: /
    destination: /var/www/books
```

... of course initially it wasn't obvious, since I was originally passing `source: /index.html` and staring blankly at the `ls` output when I had no files over on the machine...

# Deploy to staging
The code is now getting to the machine and I'm working on getting the env right:

Since this is turning out to be such a long thing, I feel the need to remind you this isn't a tutorial but more of a log of my processes as I go through the code. At this point, I started the post 6 days ago. 

Lots of issues appear since I'm not just using new stuff but reusing (like my staging machine). This is what actually happens IRL. You don't always get to `do-release-update`, instead:

## Start to update php

Take a quick look [here](https://www.rosehosting.com/blog/install-php-7-1-with-nginx-on-an-ubuntu-16-04-vps/) and write some commands...
```
sudo add-apt-repository ppa:ondrej/nginx-mainline
sudo apt-get update -y
sudo apt-get install php7.2 php7.2-curl php7.2-dev php7.2-gd php7.2-mbstring php7.2-zip php7.2-mysql php7.2-xml php7.2-opcache php7.2-redis php7.2-fpm php7.2-imap php7.2-mcrypt
sudo apt-get install php7.1 php7.1-common php7.1-cli
sudo apt-get install systemd
...
```

... take a moment to reconsider choices in general... decide to `do-release-upgrade` up to ubuntu `16.04` then ...

## Install `php7.2` over ubuntu 16.04

Guide used [is here](https://thishosting.rocks/install-php-on-ubuntu/)
```
sudo su
apt-get update && apt-get upgrade
add-apt-repository ppa:ondrej/php
apt-get update
apt-get install php7.2
php -v
apt-get install php-pear php7.2-curl php7.2-dev php7.2-gd php7.2-mbstring php7.2-zip php7.2-mysql php7.2-xml
```

...Get all the way to [Configure webserver](#configure-webserver) to realize I missed installing `php-fpm` so...
```
apt-get install php7.2-fpm
sudo service php7.2-fpm status # test it worked
```
* Error++

Rerun the deploy, get a new error. This time it's 
```
Script at specified location: scripts/start_application.sh run as user ubuntu failed with exit code 255
```
Problem was a wrong path in `start_application.sh`. 

* Useful locations for debug/logs

```
/opt/codedeploy-agent/deployment-root/
/var/log/aws/codedeploy-agent
```

## Setup the project

I broke this up into small chunks for easy digestion. Get it, `cos this is food for thought ☜(˚▽˚)☞

### Configure mysql
This is done once at the beginning, using this sql script:
    
```
CREATE DATABASE IF NOT EXISTS `books` COLLATE 'utf8_general_ci' ;
CREATE USER 'books'@'localhost' IDENTIFIED WITH mysql_native_password BY '***';
GRANT ALL ON `books`.* TO 'books'@'localhost' ;
FLUSH PRIVILEGES ;
```
Then I test the connection locally `mysql -ubooks -p***`. 
      
_By the way_, I still remember the hours I spent long ago, thinking there was supposed to be an empty character between `-p` and the actual password.
       
### Configure webserver 

* Config file

Create the file `/etc/nginx/sites-available/www.books.lpgfmk.xyz` and make it look similar to this.

```
server {
        listen 80 ;
        listen [::]:80;

        root /var/www/books/public;
        index index.php index.html index.htm;
        access_log /var/log/nginx/books-access.log;
        error_log /var/log/nginx/books-error.log;

        server_name books.lpgfmk.xyz;

        location / {
                try_files $uri $uri/ /index.php$is_args$args;
        }

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        location ~ \.php$ {
                try_files $uri =404;
                fastcgi_split_path_info ^(.+\.php)(/.+)$;
                fastcgi_pass unix:/var/run/php/php7.0-fpm.sock;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                fastcgi_index index.php;
                include fastcgi_params;
        }

}

}

```

* Enable the config and reset the server
    
```
ln -s /etc/nginx/sites-available/www.books.lpgfmk.xyz /etc/nginx/sites-enabled/www.books.lpgfmk.xyz
sudo service nginx configtest
sudo service nginx reload
```

* Route53 config DNS 

Add the IP of the machine to a new A record for `books.lpgfmk.xyz`. Later in the [letsencrypt](#from-gitlab) section, I will add a second A record pointing to `www.books.lpgfmk.xyz`. 

* Test config

Browse to `books.lpgfmk.xyz` and expect errors from Laravel. We fix these next

### Configure `.env`
This is done only once then not touched again. Variables here should not live in source control.
* Copy the local `.env` to the remote
* Setup permissions
```
cd /var/www/books
sudo chown -R www-data: .               # webserver user should own the files
sudo chmod -R 0777 storage              # these need extra access
sudo chmod -R 0777 bootstrap/cache      # these need extra access
```
_By the way_ I can't remember why I set the permissions to `0777`, anyway they don't stay like this since they're [overwritten at deploy time](https://bitbucket.org/anrei0000/hiring_books/src/1dbb9d6e9cd0ddd62b5084c4fe2e257d58ea0033/scripts/start_server#lines-12).
* set key `sudo php artisan key:generate`
* set remaining variables... dunno which ones `¯\_(ツ)_/¯` try and get all of them
* test everything works by going to `books.lpgfmk.xyz`

## Test deployment
Getting ready for the final action but still keeping it cautious. Bitbucket only has 50 free minutes of deployment time a month you know...  

### From my local machine
* Error++

```
scripts/start_application.sh run as user www-data failed with exit code 1
```

Debug this by running the commands in the file from the remote machine as well.

This way, surprise! actual error is `[2002] php_network_getaddresses: getaddrinfo failed:`

The problem was a missing variable in `.env`. Damned if I can remember all those variables (ಥ﹏ಥ)

* Error++ when deploying by push to master

```
InstanceAgent::Plugins::CodeDeployPlugin::CommandPoller: Error during perform: InstanceAgent::Plugins::CodeDeployPlugin::ScriptError - Script at specified location: scripts/install_dependencies.sh run as user www-data failed with exit code 1
```

* Error++ after removing the `install_dependencies` script  

```
"Script at specified location: scripts/start_server.sh run as user ubuntu failed with exit code 126\"
```

Tested out changes to file permissions from [here](https://laravel.com/docs/5.8/installation) and [here](https://stackoverflow.com/a/37266353/1486950). 

Tested out removing the files and file contents from the scripts.

Tested out doing 
```
sudo rm -rf /opt/codedeploy-agent/deployment-root/ff935564-c1b7-4758-9d29-acf5dd657e21/
sudo service codedeploy-agent stop && sudo service codedeploy-agent start
``` 

Reread about [lifecycle events](https://docs.aws.amazon.com/codedeploy/latest/userguide/reference-appspec-file-structure-hooks.htmlhttps://docs.aws.amazon.com/codedeploy/latest/userguide/reference-appspec-file-structure-hooks.html) and [error codes](https://docs.aws.amazon.com/codedeploy/latest/userguide/error-codes.html)

Tested out loading files from the beginning from [this repo](https://bitbucket.org/awslabs/aws-codedeploy-bitbucket-pipelines-python/src/master/).

Found a new error++ just above the previous one in the log:
```
The CodeDeploy agent did not find an AppSpec file within the unpacked revision directory at revision-relative path "appspec.yml"
```
Tested the actual script on [http://www.yamllint.com/](http://www.yamllint.com/).

* Fixed locally by:

    * [Reinstall codedeploy](#just-in-case-uninstall-codedeploy)
    * 
    ```
    sudo cp /var/www/books/.env /var/www/
    sudo rm -rf /var/www/books/*
    ```
    * Run the deployment from git revision `95cab152209e6ac5ee268f76a569e73ffe570c69`
    * 
    ```
    sudo cp /var/www/.env /var/www/books
    ```
    
    * Rerun deployment with the exact same content as before: Deployment succeeded
    
    * Rerun deployment and change the contents of a file: Deployment succeeded
    
    * Reboot the machine and rerun deployment: Deployment succeeded
    
    * Create a conflict in the file above (change it on the machine and locally) and rerun deployment: Deployment succeeded
    
    * Set ownership then rerun deployment
    ```
    sudo chown -R www-data: /var/www/books
    ```
    : Deployment succeeded

* Fix the deployment

Error++: the `books.lpgfmk.xyz` site doesn't load, so I'm listing things I tried other than nginx related stuff.

Change the `appspec.yml` file by adding the `AfterInstall` step: Deployment succeeded

Change the `appspec.yml` file by updating the scripts names, commenting out `install_dependencies` and `stop_server` contents: Deployment Succeeded

Ran `composer update` on the machine.

Ran `sudo systemctl status nginx` and got 
```
nginx.service: Failed to read PID from file /run/nginx.pid
```
Fixed this by doing this [workaround](https://bugs.launchpad.net/ubuntu/+source/nginx/+bug/1581864):

```
sudo mkdir /etc/systemd/system/nginx.service.d
printf "[Service]\nExecStartPost=/bin/sleep 0.1\n" | \
sudo tee /etc/systemd/system/nginx.service.d/override.conf
sudo systemctl daemon-reload
sudo systemctl restart nginx
```

Fixed by loading the site in mozilla browser... apparently there is some caching issue in my local chrome which I'll just ignore for now.

### From gitlab
Push the project to master at `43601d65dc3db0c2b697933ce0408b23a21c608d`.

* Add the `composer install` command to deployment script.

* Install certbot by [following this tutorial](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-16-04)
    * Add A records for `books.lpgfmk.xyz` and `www.books.lpgfmk.xyz`
    * Steps  
    ```
    sudo add-apt-repository ppa:certbot/certbot
    sudo apt-get update
    sudo apt-get install python-certbot-nginx
    sudo systemctl reload nginx
    sudo certbot --nginx -d books.lpgfmk.xyz -d www.books.lpgfmk.xyz
    sudo certbot renew --dry-run
    ```
    
    * _By the way_ here's how to delete an extra certbot site
    ```
    rm -rf /etc/letsencrypt/live/${DOMAIN}
    rm /etc/letsencrypt/renewal/${DOMAIN}.conf
    ```
  
* Install `npm`
    * Steps
    ```
    cd ~
    curl -sL https://deb.nodesource.com/setup_6.x -o nodesource_setup.sh
    sudo bash nodesource_setup.sh
    sudo apt-get install nodejs
    ```
    This isn't really mandatory but nice to have for now.

# Conclusions
Oh the joy! It feels like forever since I wanted to implement this but just didn't have the time for it. Now that I also managed to put it in writing it's double the excitement since next time it'll be easier to implement again.

In case you missed it, here is the [deployed project](https://books.lpgfmk.xyz) and here is the [bitbucket repo](https://bitbucket.org/anrei0000/hiring_books/src/master/) for this post.

I had a lot of fun, pain and insight throughout this entire process. And will be definitely using this for larger projects where this comes in handy.

Now, all that's left for me to do is grab a drink and marvel at all the typos above.

**For more discussions on this topic I encourage you to join [my facebook group](https://www.facebook.com/groups/477320829754434/).**

Hope this helps you as well as it did me! Thanks so much for following along and please come back often for new content. 

# More reading
* _Next actions_:
[The great CEO within](https://docs.google.com/document/d/1ZJZbv4J6FZ8Dnb0JuMhJxTnwl-dwqx5xl0s65DE3wO8/edit#heading=h.zdta3gwko2l) is a wonderful resource. You may read more about this in Chapter 3: Getting things done.
* [Github now supports CI/CD free for public repos](https://github.blog/2019-08-08-github-actions-now-supports-ci-cd/)
* [CodeDeploy ApplicationStop Lifecycle](https://github.com/aws/aws-codedeploy-agent/issues/80)