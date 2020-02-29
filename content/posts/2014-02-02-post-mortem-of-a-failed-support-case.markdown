---
title: "Post-mortem of a failed support case."
date: 2014-02-01
---

Every now and then I check the link reports for Project FiFo to see what people think and write about it. Recently I stumbled about [an article](http://consulting.miracleas.dk/blog/post/2014/01/17/Quick-and-Dirty-spawned-LightlyCloudy!.aspx) that oddly enough made me both proud and sad. It actually was a rather negative one, which is a shame, but on the other hand a project isn't mature until people care enough to complain.

Yet even so it would be very easy to cast this aside as a 'success' in a strange manner, it still bothers me that someone is upset enough with FiFo to spend his time writing a longish blog article and write their own management software. I do take great pride in the fact that we do our best to have an outstanding documentation and a good support for users of FiFo, and I dare to say so does anyone else who is involved in the project.

So armed with the full history of events  plus the additional background from the [article](http://consulting.miracleas.dk/blog/post/2014/01/17/Quick-and-Dirty-spawned-LightlyCloudy!.aspx) mentioned above I want to try a post mortal analysis of this support case, what went wrong and how it could be avoided. I obviously will try to be as impartial as I can be but in the end I can only look at what I see. Hopefully there is a lesson to be learned from this so next time things went smoother.


As a disclaimer: the problem is still a mystery and will probably remain so forever. Also all documentation linked is from the time the bug was filed (thanks for versioning) to give a fair picture of what was available back then.

# Act 3 - the mailing list
The history of this case starts a bit into the history of the whole story, but for now lets skip the first part and look at the first time we (as in the FiFo team) came aware of the problem: [A mail to the mailing list and the conversation following it](https://groups.google.com/forum/#!topic/project-fifo/MZJYOs-CtP8).

Sadly the initial mail gives every little to no information on the problem aside that 'not firing on all cylinders/Suddenly FiFo stopped working all by it self'. I must admit that for me such a thing is a big red flag that the person on the other end does not care much or is not willing to invest energy into finding the problem, looking from today and after reading the articles on Gordon's blog it might as well just have been the kind of humor used, we will never know.

But fortunately I am not the only one on the mailing list and others are slower to judge and have more patience and Mark jumped in to help and tried to help, starting to ask for additional information that were missing from the original mail. After the information not showing any additional evidence that could lead to an easy conclusion Marks ask for him to take a look at [FiFo's Problem Checklist](http://project-fifo.net/pages/viewpage.action?pageId=8126470) and come to the IRC channel to help further.

## Act 3 conclusion

Mark replied to the initial mail within less then an hour which is a truly amazing response time that can put most commercial support to shame request to go through the checklist and join the channel for a more direct help is a very good approach.

Since I can actually look into my head I shed a bit more light on my take at the point, I decided to opt out of that request at that moment since Mark seemed to have it covered and the tone of the mail already made me sort it in a bucket of 'going to be annoying'. So I probably should not judge as fast.

To receive the best possible support there are some things Gordon could have done, the initial mail could have contained more information, the tone could have been different — jokes often communicate badly over written text, and following Marks request to go through the checklist had probably helped too if only to get some additional facts for later.

# Act 4 - IRC
Marks suggested to swap to IRC for a quicker communication, we all hate mail ping pong I guess. Fortunately we [keep logs](http://echelog.com/logs/browse/project-fifo/1382914800) of the FiFo channel to make it easy to google for known problems that were discussed before — in this case it allows us to look at the conversation between Mark (aka trentster) and Gordon (g-flemming).

This is pretty uneventful mark asks a few more questions to rule out common errors none of which seem to be the cause. At this point Mark asks to escalate the issue and file a ticket with the logs and offers the suggestion to move to a newer version of FiFo (the development build at that time) along with [the manual containing steps necessary to do this](http://project-fifo.net/pages/viewpage.action?pageId=7176245). The reasoning being that the development build is quite well tested with multiple people (including Mark and myself) running it.

## Act 4 conclusions

Things are still pretty well at this point, our escalation process seems to work wonderfully and the resources of the documentation hold a lot of valuable information.

This is where the story ends for Mark and to all honestly he did an astonishing job to support here and escalate when he ran out of ways to help.

On the channel Gordon mentions the first time that he is running FiFo with customer on it this indicates some urgency on the matter and at least I missed that line. Sadly so he did not seem to have went through the troubleshooting steps or read the update documentation.

The lesson to learn here for us is to either ask people to include an IRC log in a new ticket or do it ourselves after they create it, this might have added some additional info to the ticket that was not present when it was created.

# Act 5 - The ticketing system
Now this is where my part in the story starts, just as before I'll try to give some additional insight on what I was thinking at this time to shed a bit of light on my end - I obviously can't do the same for Gordon.

As asked by Mark Gordon created a [JIRA Ticket](http://jira.project-fifo.net/browse/FIFO-142) with the logs. When seeing the ticket I had already read the mailing list but not the IRC backlog (I don't always do that since I don't want to spend too much time reading old things that might not even affect me). From the ML I have already an unhappy feeling towards the issue as it looked to me that Gordon was not willing to put much effort into resolving the issue.

Non the less I take a look at the logs and trie to pice together what exactly happened, up to this date I do not know what it was. The suggestions were pretty close to what Mark suggested earlier, a problem with too little memory or the filesystem the error in the logs `einval` hinted to some issue with a POSIX filesystem call.

And at this point things are going wrong, lacking the information that there are production users on the system I make the mistake to judge the issue as non urgent especially after the full reply from Gordon takes a day. At that point I pretty much stop caring about the issue since I feel neither does Gordon (which is a grave misjudgment as it turns out). I revisit the issue only one week later at which point I now can only assume Gordon had given up and in a last attempt to provide a direction suggest looking at FS errors, missing the question asked in Gordon's reply.

Gordon makes the mistake of not reading the documentation Mark gave him earlier or the question he posted next day would have already been answered and probably waiting a day with replying to the question.

## Act 5 conclusion

I need to stop drawing conclusions so quickly and read bug requests more carefully or I would not have missed the migration question in the bug report. It would also be worth a try not to treat less urgent tickets with less care, it sucks to have tickets open the goal should be to close (as in resolve) then as soon as possible even if they seem not urgent — this applies especially to bugs.

Gordon could have made his own life a lot easier by including more information in the ticket, noting that it is an urgent issue, including the history of what he already did to debug would have both helped a lot. In addition to that actually reading the documentation Mark provided would have answered the migration question beforehand.

Mark while out of the picture already could have included the chat history in the ticket when seeing that Gordon did not — this admittedly is asking a lot.


# Act 0 - how everything started
I know this is the wrong order, but Star Wars got away with it too. Now after the fact I know more then I did before. Reading Gordon's blog posts shed some light on the history and I think this is where actually things started to go wrong.

I am glad for every person choosing to try out FiFo, it means people put trust in what we've build and that is a really cool thing. But please if you want to use something in production inform yourself ahead of time. Don't put yourself and your customers at risk by blindly running into things.

Talk to us, even before you start deploying! We know FiFo inside out, everyone in the FiFo team is running it themselves either for fun, for profit, for testing or for all three things. The channel is helpful too, there are more people outside the core team on the channel too who will gladly share their stories.

To put this straight, you'll not only get the software for free you will even get some "consulting" tossed in the mix for not anything more then just asking! That is of cause in a sensible limit and given we have time, but there is always half an hour to spare here or there.

## Act 0 conclusions
Deploying in a single node for a production system was not the best move, FiFo clusters for a reason and a distributed setup has many advantages over a single node. Probably some of the config settings could have been tweaked for a better user experience. It might have even made sense to run on dev instead of release or at least to switch early.

Had we known the surrounding circumstances helping might have been a lot easier.

# Act 1 - be active in the community
There is a huge advantage to this. And I don't even mean the fact that the community grows and everyone benefits. Being present in the channel and occasionally talking to people helps you to stay in the loop, know what is going on and what other people face for problems or find for solutions.

It also gives the benefit to influence the course of the project, a lot of the features were thought up and discussed within the community. And last but not least being active might actually end up in helping others, which will make the community as a while stronger.

## Act 1 conclusion
I admit I am totally biased here. Everyones time is limited. When I have to chose to help a stranger I have no idea who it is or someone I know and has contributed in one way or another to the community I will pick the community member every time.

I can only talk for me here but I know for a fact that if Gordon or one of his colleges had been around in the channel and were a known face I would have taken the problem more serious. That said I have no idea if that is good or bad that I put community members before strangers.

# Act 2 - read the fantastic manual
As Gordon points out FiFo not a trivial application. And he is entirely right, it is not, and there are may reasons for that which I am not going to argue here if that is a good or bad thing I will simply state that I have thought long about every choice I did in FiFo and claim them to be sound.

But we are well aware that it is not a simple pice of software like a editor or something, that is why we put a lot of time and effort in providing a manual and an extensive set of informations surrounding it.

We have a fully fledged manual, guides for Installation, Migration, and update. Best practice articles for Scaling, Networking and clustering. Checklists for problems and known issues. A list of terminology, information about our versioning system, recorded trainings (admittedly not much) and videos on usage.

For people interested in developing we have API documentations, starter guides, a documented build process. We have a list of internal libraries, and specifications on data structures. A guide to plugins, the messaging system and how to write plugins.

And last but not least we even have [a page in which we explain how to best submit a bug](http://project-fifo.net/pages/viewpage.action?pageId=4194636)

## Act 2 conclusion
Reading those documents this would probably have saved a lot of time and pain and the fifo team some work. This is a reoccurring problem that we sadly see way too often  - if anyone has suggestions how to encourage users to read manuals please share the holy grail.

The documents would have given good advice on how to set up a redundant fifo, how to check for problems and perhaps most importantly how to properly report a bug.


# The Rant
Given I spent the last two years of my life working on Project FiFo I feel I am entitled to this. Bad bug reports are a pet peeve! And this was a prime example of one.

All the right signs were there, a entirely nonsensical title the catch phrases 'it stopped working' and 'nothing had changed'. I can't say if this is the one in a million where that was actually true but in all my time in IT I have never seen those words to be correct, nor have I ever heard from someone else that saw the mystical situation of something just stopping to work.

There was no usable information in there, no sign of interest to actually help the process of finding the root cause. Just because FiFo is free and there is no charge in using it does not mean my time is worth any less then yours, gladly help you with a problem but I'll expect some engagement in return. I do not like to have my time wasted by having to pull every but of information out of someones nose.

Bottom line is: If you don't care enough about your problem to put some effort into getting help I will not care enough to help.