---
title: "Five Years of Minetrack"
date: 2020-07-11T14:42:45-05:00
tags: ["minecraft", "minetrack"]
summary: "A celebration of Minetrack's 5th birthday: reflections on the project's history and next steps forward."
---

Over five years ago I quickly wrote and glued together [Minetrack](https://minetrack.me) (originally "FastTrack" 🙃). It was effectively an auto refreshing web version of the [Minecraft multiplayer menu](https://minecraft.gamepedia.com/Tutorials/Menu_screen#Multiplayer) used to display player counts.

{{< figure src="/Minetrack_images/early_version.png" attr="An early (redesigned) version of Minetrack, shortly after release" >}}

**But why?**

It had a simple purpose: in an era of frequent [DDoS attacks](https://www.cloudflare.com/learning/ddos/what-is-a-ddos-attack/) on Minecraft servers, Minetrack offered an easy way to correlate player count shifts between competing Minecraft servers potentially suffering network issues.

Minetrack was not a new idea. Several sites already existed to track Minecraft server player counts, such as [Top MC Networks](https://web.archive.org/web/20130718164208/http://www.topmcnetworks.com/). However they focused on historical trends (5 minute update intervals) whereas Minetrack offered _real-ish_ time 3 second update intervals. To help differentiate Minetrack (and avoid the nightmare of Javascript graphing libraries), I even refused to add a historical graph for over a year after release. 🤦‍♀️

{{<figure src="/Minetrack_images/later_version.png" attr="A later version of Minetrack featuring the long awaited historical graph" >}}

**Well...**

Minetrack was first and foremost, a tool designed for _me_. I offered it to others, and [open sourced its code](https://github.com/Cryptkeeper/Minetrack), but the design, feature set, and Minecraft servers it watched, were always reflective of my use case.

Surprisingly, people embraced its messy codebase and limited functionality. Soon after, people began adapting Minetrack to fit their own use cases.

Some people deployed their own Minetrack instance with a custom list of watched Minecraft servers. ⬇️

{{< figure src="/Minetrack_images/minetrackdotbreakingmcdotnet.png" attr="A Minetrack instance with a custom list of watched Minecraft servers" attrlink="https://minetrack.breakingmc.net" >}}

Some people redesigned it.

{{< figure src="/Minetrack_images/trackedserversdotcom.png" attr="A redesigned version of Minetrack seen at trackedservers.com" attrlink="https://trackedservers.com" >}}

Other people used it to monitor their internal infrastructure instead of public facing Minecraft servers.

{{< figure src="/Minetrack_images/statsdotnethergamesdotorg.png" attr="A Minetrack instance being used for internal infrastructure monitoring" attrlink="https://stats.nethergames.org" >}}

Finally, some weird people decided to rip out the backend and plug in their own, repurposing the frontend.

{{< figure src="/Minetrack_images/hytrackdotme.png" attr="A repurposed copy of the Minetrack frontend for monitoring Hypixel minigames" attrlink="https://hytrack.me" >}} 

Early on, there was even a (briefly working) Chrome extension built by a friend.

{{< figure src="/Minetrack_images/chrome_extension.png" attr="A Minetrack Chrome extension" attrlink="https://chrome.google.com/webstore/detail/minetrack-chrome/jliecigdlnddhdjddngpeknemdlgdkli?hl=en-US" >}}

Slowly, various community bug reports and pull requests began to pop up. 

I started to relax my tight grip on the project and focused on welcoming these incoming changes. Piece by piece Minetrack evolved from a quick hack into something useful for the larger community, even if it was just to see who's fault a lagging connection was.

**A Brief Timeline**
* Jan. 4th 2015: First commit! 🎉
* Jan. 19th 2015: Added [Mojang API](https://wiki.vg/Mojang_API) and service status indicators.
* Jan. 31st 2015: Added the ability to add `mc.minetrack.me` to your Minecraft multiplayer server list.
* Jul. 26th 2015: Original Java codebase is deprecated and a [Node.js](https://nodejs.org/en/) version is started.
* Nov. 2nd 2015: [Minecraft "Pocket Edition"](https://minecraft.gamepedia.com/Pocket_Edition) support added.
* Nov. 25th 2015: Community feature: [Minecraft server type indicators](https://github.com/Cryptkeeper/Minetrack/commit/671c410a682b574db6f82d9f33b37d3fd5665a87).
* Nov. 30th 2015: [Player count summaries](https://github.com/Cryptkeeper/Minetrack/commit/dd8f5ec10d332aeecda5a614c0f281231a920fd4).
* Dec. 10th 2015: [sqlite3 database logging](https://github.com/Cryptkeeper/Minetrack/commit/01f977b16e8892021e6d8cb34f75264bd6827a39).
* Dec. 18th 2015: [Historical graph is added (with controls!)](https://github.com/Cryptkeeper/Minetrack/commit/8f66475a28ee9ce32844d5621f90353bc18baa61)
* Feb. 23rd 2016: Added "categories" to organize the growing list of Minecraft servers.
* Mar. 1st 2016: Community feature: [Minecraft protocol version compatibility tracking](https://github.com/Cryptkeeper/Minetrack/commit/43c284aa8a36566a43699b73ab3b836b76ffc1fd).
* Jun. 8th 2016: Community feature: [SRV record unfurling](https://github.com/Cryptkeeper/Minetrack/commit/0034d7fc036867474cf587ec251dbe3327bbe37b).
* Mar. 11th 2017: Added player count record tracking.
* Mar. 11th 2017: [Added player count market share "percentage bar"](https://github.com/Cryptkeeper/Minetrack/commit/f722f5295a9264bd64217ee3141f7a5692d1e554).

And with the exception of little tweaks and fixes, that is how Minetrack sat for the next two years. In fact, there was a total of **5** commits during 2018 and 2019 combined - with 3 being community contributions!

![img](/Minetrack_images/2018_2019_commits.png)

What happened? Well, _nothing_.

Minetrack quietly ran without issue on its tiny $5/month [DigitalOcean droplet](https://www.digitalocean.com/products/droplets/), serving its purpose. Focused on [Hytale](https://hytale.com), I had forgotten about Minetrack and considered the project unsupported. It had been abandoned to run as long and as well as it could without intervention.

What was Minetrack doing during this abandonment? Serving terabytes of bandwidth* monthly to thousands of daily visitors. Even abandoned, Minetrack was still useful, which was more than I could have ever expected of it.

_*Admittedly, some of that bandwidth usage is likely due to its original, unoptimized protocol._ 😄

**So, what's next?**

Starting in 2020, several new versions of Minetrack have been released. These updates have focused on patching several long standing bugs, offer a host of optimizations, and a much needed facelift.

You can consider this a spring cleaning. 🧹

{{< figure src="/Minetrack_images/minetrackdotme.png" attr="The latest version of Minetrack as seen at minetrack.me" attrlink="https://minetrack.me" >}}

My goal is to continue shipping some more minor updates focused on user experience improvements and bug fixes.

Besides code updates, I've released over 4 years of Minetrack's databases freely into the public domain at [data.minetrack.me](https://data.minetrack.me/). My hope is to help kick-start some long term analysis of Minecraft's multiplayer player counts.

**In summary**

Completely against my original expectations, Minetrack has celebrated its 5th birthday and I'm excited for its next 5 years. A big thank you to those of you who have helped shaped it over the years.

Finally, an immense level of credit is due to [Minetrack's open source contributors](https://github.com/Cryptkeeper/Minetrack/graphs/contributors), and in particular [@hugmanrique](https://github.com/hugmanrique), who have collectively helped fix bugs, add features, and maintain the project. Thank you! ❤️
