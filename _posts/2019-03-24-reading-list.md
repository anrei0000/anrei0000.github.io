---
layout: post
title: "Weekly knowledge dose"
date: 2019-03-24
---
Hello and welcome to my weekly archive of applied knowledge I loved going through. 

Here's this week's top 5 pick in the order I found them:
- A fun technical read! [Chris Wellons on nullprogram.com](https://nullprogram.com/blog/2019/03/22/) wrote [Endlessh](https://github.com/skeeto/endlessh) a tool for capturing your port sniffing bots for a measured 1378 clients up to 20 hours. 
- [Jessica Livingston is paying for 40 women's coding bootcamps this summer](http://foundersatwork.posthaven.com/women-learn-to-program-this-summer). She's a cofounder and partner at Y Combinator and this seems like an awesome chance for a head start in IT. You can register [here](https://www.lambdaschool.com/summer-hackers/) until 1.4.19 for an on-line summer duration 15 week program for *US based residents only* with [Lambda School](https://lambdaschool.com/). 40 applicants will get picked to have their brains expanded with everything from startup knowledge to dev work from the confort of wherever they choose to spend their summer. 
- Here's some extra proof that [exercise is good for you and your brain](https://www.outsideonline.com/2186146/your-brain-exercise). Read this in case you need a "There's a study on this..." argument. Basically, experimental proof is there that after exercise our senses are heightened, our "perceptions are sharper", which sounds like good news when you feel like needing an extra kick during exams or deadlines.
- [HTML5 periodic table](https://websitesetup.org/html5-periodical-table/) seems a good place to go and easily find stuff I should be using just for HTML.
- [Nikita Voloboev](https://twitter.com/nikitavoloboev) wrote [Everything I know](https://wiki.nikitavoloboev.xyz/) and my mind is blown. I've been working on a personal backup of my mind but this is next level :) His notes grew into his Git Book but he doesn't stop there. The guy [automates his tasks](https://wiki.nikitavoloboev.xyz/macos/macos-apps/2do), [wrote an Alfred workflow](https://medium.com/@nikitavoloboev/writing-alfred-workflows-in-go-2a44f62dc432) which he uses himself, and basically tries to offload information into processes with the help of IT. Loved it!

Also my fun problems solved for the week:
- `Error starting userland proxy: mkdir /port/tcp:0.0.0.0:8000:tcp:172.30.0.3:80: input/output error`
The final fix is to [disable "fast startup" in power settings](https://stackoverflow.com/a/47818614). The issue was that a windows update enables a fast startup feature which misbehaves with docker. So my containers wouldn't start after the laptop wakes from sleep. Yuck.
- How to connect an IDE to my docker DB instance
Since I'm learning about docker and how to use it properly I'm still having issues with really simple stuff. I don't want to use phpmyadmin since I really enjoy the productivity boost from DataGrip. So all I had to fix was to add port forwarding to my db service in the Dockerfile then connect using `localhost` and the new port.
- How to edit video content from my HERO4
I never tried this before but I really enjoyed learning about video editing. My main resource was [this guy on youtube](https://www.youtube.com/watch?v=NXnBCkF9jUQ) and from there I found out about [DaVinci Resolve](https://www.blackmagicdesign.com/products/davinciresolve/) which I played around with. I will keep adjusting things but my workflow was to go out, take some pictures and videos at different settings, then go home and use DaVinci to try different settings on them. It's super easy to do, and looks great!

So, I hope you enjoyed the reads and have a great week!