---
title: "Happy birthday Project FiFo"
date: 2013-05-06
tags: ["fifo"]
---

Some might know it, some might not and some might not care but for what it's worth I'm the author of [Project-FiFo](http://project-fifo.net) (or most of it) and today is Project-FiFo's first birthday (since the domain registration) and I want to take this chance to look back to the past year and reflect, say thank you to all of you and take a look in the future.

When I started Project FiFo a year ago it was more of a tiny hobby project and I could have sworn it would stand in row with all the other little open source projects no one would ever give a damn about. I really could not have been more wrong, what started as a [few lines of clojurescript](https://github.com/project-fifo/vmwebadm) has grown to a beat of project with thousands of lines of code, a ever growing and incredible community (the project page gets between 2.5 and 3 thousand visitors a month by now and constantly more then 20 people in the irc channel) and a totally enthusiastic team!

## Thank you

With a year gone it is about time to call out a few people and say 'thanks' because without their time, effort and work FiFo would not exist and in the day to day business of killing bugs, adding features and contemplating world domination it's easy to forget this.

I want to start with [Mark Slatem aka trentster](http://blog.smartcore.net.au/) author of the SmartCore blog and FiFo's [number one](http://ukiahman.tripod.com/Ryker_7.jpg). He was pretty much the first person looking into FiFo and has sticked around till now going from first observer to tester, [writer](http://project-fifo.net/display/PF/The+FiFo+Manual), helper and most of all a good friend.

[Deirdré](http://www.beginningwithi.com/) the Joyent community manager for SmartOS. Solaris and with that SmartOS is a underdog, and I'm sure without the incredible brilliant community it would have been doomed from the start. But it is not, and in a good part thanks to Deirdrés effort to shape the community and make it part of the ecosystem instead of a second class citizen.

Killphil author of jingles, FiFo's web UI, it is amazing he popped up one day, out of the blue saying 'hey I've played a bit with improving your UI' and put jingles down with became the official UI within matters of days. Sadly I could not get rid of him again since then so you all have to live with him adding [crazy new features](https://monosnap.com/image/QNxrmvEecVoly173dt5OOrVgc).

[Joyent](http://joyent.com) as a whole for open sourcing SmartOS and making it better and better. Without SmartOs there sure would be no FiFo and I'd be stuck with Linux/KVM, which would be not too much fun.

[basho](http://basho.com) who open sourced riak_core and riak_test which make fifo so much more incredible and provide me with free T-Shirts (I got four by now but please don't tell them or I might not get any more).

Every single person using FiFo, it's amazing to see how well the project is received, hear the feedback. Thanks a lot for all the help with the little things, for putting up with the occasional bugs and bearing with the time it might take to fix them or add new features.

## Looking back

I've been working on FiFo for a year now, well first attempts counted a bit longer even, and without exaggerating this was the most amazing year in my life. It has been a blast, I've learned tons of things, both technical and socially, meet some of the most amazing people I can think of and honestly never have been this happy before.

The whole thing started when I wanted to share a co-located server with some friends who are kind of consoleophobe and I wasn't happy with the approach to give everyone root access. So a solution had to be found but vanilla SmartOS provided none, SDC was making no sense with a single node and too expensive for a hobby system. Everything else was simply not Solaris, period. Adding to it that Deirdré showed me the community, randomly answering a question on twitter with a hint to visit the channel - which was was incredible surprising after experiencing some of the Linux community… really there was no chance in hell ending up with anything but SmartOS.

But all in all there was no virtualisation solution that suited what I wanted, not even if I had taken cost out of the equation. And since I refuse to swing the white flag and surrender to something I don't like the only wan was: build one! (also I'm crazy about challenges and seeing how far I can push things ;) And after

That sums up how the whole thing started, with a little nodejs/clojurescript application that could be used with sdc-* commands over http. But that did only work with a single host, not that I had more to serve but it looked kind of clumsy and unprofessional for a cloud operating system like SmartOS so wiggle was born as kind of a broker in front of multiple vmwebadm (man the name was horribly boring). And from there on it kept growing and growing.

Now over the last year of work, and lots of lots of input from the community the one badly named program has become 5 services, three of them distributed via riak_core, and a HTML/JS UI on top of that, that most importantly, all have very cool names.

All the technical things aside, running an open source project where people get engaged is a fascinating experience, there is so much to learn I would have never dreamed of, so much to take away from the situation that helped me understand the problems and inner working of teams, people, projects 
better. That alone was worth every second of time invested.

## Dogs and funky names

Now before we go on I share something that was asked a few times now: why the crazy names and obsession with dogs? 

So the story starts with naming the first component, which back then was wiggle (after being very disappointed with my name choice for vmwebadm). Wiggle was the component that gave a unified interface to multiple hypervisors and in the SmartOS channel I had heard that Joyent called it's thing 'headnode', but for FiFo the goal was never to clone something that already existed and I wanted to make a point of it so here is how the thought chain went: head -> tail, node -> nod -> wiggle, tail&wiggle -> dog.

Now I had a promise to keep, back when I was younger and my brother was even younger (since he is my little brother) he got a pet (as in not real) dog, and I had just learned with a FiFo (First in First out) queue is, I found that Fifo is a amazing name for a dog, so I talked my brother into naming his pet dog Fifo telling hime that if I ever had a dog I'd name it Fifo too, that said, I'm a man of my word even if it takes over something like 15 years to make good on it.

All other names just followed the same naming scheme, expect jingles but then again I did not name it myself. That said Fifo, the dog, is still with us, he is sitting on my window board drying from taking a shower earlier today to get cleaned up for it's birthday!

As a nice side effect it is great for people to remember things that are named so silly as FiFo's components!

## Looking ahead

To be honest I feel that even after a year FiFo is still in its infancy, don't get me wrong it's quite stable and the features it provides build a very good foundation but there is so much more it can and will be!

PXE booting, integrated in FiFo, allowing to spin up new hypervisors by a click in the UI (or a call from he console) adding ipmi to the mix makes it even more exciting! Think about automatically booting a new hypervisor when the capacity reaches a certain percentage, or shutting an empty one down when it's below.

Support for Clouds spread over WAN, location awareness of VM's and Hypervisors with a notion of distance with deployment rules that take this into account (please don't deploy all my database cluster VM's on the same physical host, but don't spread them over multiple datacenters!).

Cold migration of VM's from one host to another, and putting them into cold storage / backing them up as a whole.

Well, I could go on and on rambling about crazy ideas for another thousand or so words or so but lets save this for another time. All in all I wanted to say it was an amazing year, amazing to see the community develop, seeing how FiFo gets used and I am hugely excited to see how things continue from here on! I can't wait to see the first 10+ node FiFo setup, hear what people make out of it. See a first adopted UI, people starting to build things around FiFo - there already is a [ruby implementation](https://github.com/bakins/project-fifo-ruby) of the API along with a [chef knife thing](https://github.com/bakins/knife-fifo).

So to close: Happy birthday FiFo, thanks to all of you for joining this journey and lets brace for another year of dog-named-components!