---
layout: post
title: "How to serve a Laravel project from Laradock & Docker"
date: 2019-07-16
---

So, for one of my potential clients I had to complete a test. And since I usually like having things ready for complex stuff I'll tell you a bit about my process and how I changed things to work for one of the requests of the test.

I found the laradock project a while back and have been using it for testing small projects for a while and I'm quite happy with the freedom I have for setting up most project environment requirements I could dream of. Laravel, nginx, aws, elasticsearch, jenkins, mailhog, neo4j, everything is dockerised and easily accessible for new projects. This makes me happee.

Enter test requirement: "make this work using php webserver". I'm now left scratching my head. "How can I most easily overengineer this ?", my mind quietly asks. So I get to testing. 

First here is the architecture:
* host machine running windows and docker
* laradock project is installed
* start laradock env command
```$ docker-compose up -d nginx mysql phpmyadmin redis workspace```
I actually have this in my `.bash_profile` as a separate command. This allows me to have a running environment for a basic laravel install.
* new project setup

```
# ssh into laradock workspace
$ winpty docker-compose exec workspace bash

# setup a new laravel project
$ cd /var/www && laravel new project

# serve the project as usual
# step1 config laradock nginx
# step2 config windows hosts
```

The fun was to make things working without nginx, and here is what actually was required for getting things working:

```
# edit laradock/docker-compose.yml and add port forwarding
	ports: 
		- "8080:8080"
# run the webserver from the public folder of Laravel
$ cd /var/www/project/public && php -S 0.0.0.0:8080
```

That's it :)

For testing I did:

```
# check the site is reachable
$ curl localhost:8080

# check the port is bound and forwarded
$ docker ps 
CONTAINER ID        IMAGE                COMMAND                  CREATED             STATUS              PORTS                                          NAMES
1213e3417fb8        laradock_workspace   "/sbin/my_init"          13 minutes ago      Up 13 minutes       0.0.0.0:8080->8080/tcp, 0.0.0.0:2222->22/tcp   laradock_workspace_1
```

Aaand here are other things I tried which didn't need to be done:
* add `127.0.0.1 localhost` to windows hosts - not needed
* add `EXPOSE 8080` to `laradock/workspace/Dockerfile` - insufficient because this didn't bind the port to the outside
* ran the php server using `php -S 127.0.0.1:8080` OR `php -S localhost:8080` - insufficient because this doesn't listen to the right network
* changed the port several times - when running out of ideas I just try stuff
* added `project/.htaccess` to redirect to the `/public` folder that Laravel uses - didn't actually debug why this didn't work and instead just ran the webserver from the different location (`/var/www/project/public`)
```
RewriteEngine on
RewriteRule ^(.*)?$ ./public/$1
```
* tested different `laradock/nginx/Dockerfile` configs and actually ended up writing my command for rebuilding the container

Also, here are a couple of stack overflow Q&As that got me started:
* https://stackoverflow.com/questions/28788285/how-to-run-laravel-without-artisan
* https://stackoverflow.com/questions/25591413/docker-with-php-built-in-server

Yay, now I can work on the project !! :))