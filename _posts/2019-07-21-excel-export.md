---
layout: post
title: "Excel export - Laravel, Laradock, Maatwebsite"
date: 2019-07-21
---

Yay, so I found myself with some time on my hands. Browsing jobs on upwork I found someone needing an excel export for some user data. That made me remember my last excel project and how complicated it had to be. I was a bit sad then said, "Well I'll be a rotten tomato, I'll just see how to do this now."

So this journey begins.

TL;DR: unexpected consequences of dusty devops are overcome and a proof of concept emerges

If it didn't end here for you, I'm happy to have you on board! Let's get into it.

## Install Laravel
I want a fresh version of Laravel on my Laradock instance. First things first, this time, I want to install using the laravel command (`laravel new blog`). So to install that, I move to installing `composer` but notice I'm logged in as `root` in my container. At first I just say, meh, whatever, I'll just install this way. But then my conscience comes into play, because in the words of <a href="https://getcomposer.org/root">https://getcomposer.org/root</a> we:

> "Do not run Composer as root/super user"

So now, I dunno which is the regular user for the laradock machine. I search on google and find it's `laradock` (I just couldn't remember `etc/passwd`). I search around a bit more and find out that I can actually `exec` into the container with a specific user. Neat! So, I think how can I use this in the future ? Well how 'bout a command to ssh into this container:

```
# Enter laradock workspace
function fmk-lara-ssh() {
	fmk-cd-laradock && winpty docker-compose exec --user=laradock workspace bash
}
```

OK, now I can really install laravel 

```
host$     fmk-lara-ssh
laradock$ composer global require laravel/installer
laradock$ laravel new export-excel
```

Then make it run locally in laradock:
* nginx config: duplicate the old config file and change the `file name`, `Server name`, `files location`, `logs locations`.
* rebuild the nginx container: in the host, run `fmk-lara-rebuild-nginx`
* windows hosts config: add an entry for my new site `127.0.0.1		export-excel.local`
* test this works: navigate to `export-excel.local` and see the Laravel dashboard

## Install prerequisites
A bit of googling gives me a tool I wanna try out. So, I head on over to <a href="https://docs.laravel-excel.com/3.1/getting-started/installation.html">Laravel excel install page</a>. 
I quickly check the versions and I'm good to go with the latest one.

```
composer require maatwebsite/excel
php artisan vendor:publish --provider="Maatwebsite\Excel\ExcelServiceProvider"
php artisan make:export UsersExport --model=User
```

## Export
Now for the export, I open the documentation and my eyes feast upon a 
<a href="https://docs.laravel-excel.com/3.1/exports/">5 minutes quick start</a> 
link.

Going through that I quickly do:
```
php artisan make:export UsersExport --model=User
```

I want to add the route but notice there's a typo - a missing `'`. The guy's 
site is awesome since it has a link in the page pointing right to the online
file edit in github. I add the missing character, send the pull request,
then <a href="https://github.com/Maatwebsite/laravel-excel-docs/pull/76">
he merges</a> in like 20 minutes. Awesome! Now to adding the route:
```
Route::get('users/export/', 'UsersController@export');
```

## Trouble starts
When I try to access the new route in the form of a message saying:
```
Illuminate\Database\QueryException : SQLSTATE[HY000] [2002] No such file or directory
```
First thing I do is recheck the credentials, find out that the credentials 
live in `.env` for the `laradock` project... go figure, credentials living in
`.env`... yeah, that's who you're reading, in my defense usually my brain is 
livelier.

But no dice, all is good with the credentials. I investigate further, and find I 
can't actually run `php artisan migrate` which is even stranger since... I can
connect to the DB in PHPStorm.

I think about here I am getting
```
SQLSTATE[HY000] [2002] Connection refused
```
and I don't know what to do with it. Unsettling.

Change the credentials to use the `root` account to login. Finally something 
strange to get as an error message:
```
SQLSTATE[HY000] [2054] The server requested authentication method unknown to the client
```

From here it's relatively easy to get to <a href="https://stackoverflow.com/a/53881212/1486950">this SO answer</a>
which I change slightly to 
```
ALTER USER 'default'@'%' IDENTIFIED WITH mysql_native_password BY 'secret';
GRANT ALL PRIVILEGES ON upwork_1.* TO 'default'@'%';
flush privileges;
```
For me the issue name became `caching_sha2_password new mysql 8 thing`. Also more reading
<a href="https://mysqlserverteam.com/upgrading-to-mysql-8-0-default-authentication-plugin-considerations">at this link</a>

## Trouble averted
But still no users exported! Well of course, there aren't any in storage! Just do
`php artisan make:auth`. Access `export-excel.local/register` to register a new 
user, then access `export-excel.local/users/export` and the excel export should
be comfy in your `~/Downloads` folder.

Wild and roundabout way for a "simple export"... hope you like this since my brain
doesn't wanna remember again. 

PS: Things I left out above:
* how to locally serve my jekyll instance (like <a href="https://github.com/BretFisher/jekyll-serve">this</a>)
* updates to my css during this blog writing
* "help me improve my page" made it on my `next_actions.md` list
* a new "api authentication" post will likely be next
* actual excel file output and proof this <strong>wasn't all in vain</strong>
```
1	Andrei	ciuc@ciuc.ro		2019-07-21 19:42:24	2019-07-21 19:42:24	
```
* future action might include putting up the project where you can check it out
* testing the <a target="_blank" href="{{site.github.repository_url}}/edit/master/{{page.path}}">Edit this page</a>