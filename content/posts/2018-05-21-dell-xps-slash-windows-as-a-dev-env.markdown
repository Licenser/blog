---
title: "Dell XPS/Windows as a dev env"
date: 2018-05-21
---

I've recently gotten a Dell XPS 15" 2-in-1 and started using it as a development environment for the last week. To be honest as a long term MacBook user I expected a rather disapointing experience but to my big surprise I do really like it so far. But enough of a preamble. Why I'm writing this? Because I figured that the mistakes I made, the hints I got all over the place would have really helped me if someone had collected them - so I do that now. Mind you a lot of it is not specific to the Dell and will work for every system.

## The goal

Let's set expecations. This is not about making the perfect system, after all a week isn't enough to decide what perfect is and find out all the little things that need to get tweeked. The goal here is to get a 'good enough' system that works for most everything without regretting it or missing my MacBook in as little time as possible.

## The OS

I suspect many people will be surprised but I decided to stay with windows as a operating system, partially because I've not the best opinion on Linux (which I'm not going to discuss here) and partially because I wanted to see how good the out of the box experience is. Last but not least the whole 2-in-1 experience seems to be excelently integreated with the OS and I'm a bit scared to have to fight a different OS to get it all working as it'd be two unknowns to fight instead of just one.

WSL is brilliant, I've had very little problems (read none but one super esoteric one we'll get to later) with it.

## My setp

To my great fortune I do use Eemacs which means that my environment of choice works nearly everywhere. OSX, Linux, Windows, WSL, there is an Emacs. Other then that I obviously need my Erlang, I really can't go without it ;). In addition I tossed in some rust and since I'm writing this post on the XPS ruby. The usual suspects, gcc, make, git and friends come with the terretory.

My initial attempt was to run things on windows. I tried VS code and windos emacs. Erlang, rust, git all exist as native windows binaries but the whole experience wasn't great especially since in the end it'll run on a unix or unix like system anyhow. So in the end I used WSL for help and just set everything up there.


### Ubuntu 18.04

The WSL installation I picked. I don't want to go into a distribution war and I'm sure all other distributions just work as well - pick your favourite I simply know ubuntu a bit better then I know SuSe (the alternative from the app store) and can't be bothered to find out how to add additional distributions.

Most of everything comes in `apt`, with the exception of rust which uses [rustup](https://rustup.rs) and erlang wich I fetched the [ESL package](https://www.erlang-solutions.com/resources/download.html) for to get around ubuntus horrible decision to split up erlang in multiple packages (seriously WTF?!?).


### Emacs

I use [space emacs](http://spacemacs.org/) which installes without problems on WSL, just clone the reposit and call it a day. It'll ask you for additions for languages as you open the files, a lot exist but if you do something that's more esoteric then erlang you might have to do some additional fiddling.

Here comes the first issue, WSL does not have an X server (I really hope at some point Microsoft will add one) but in the meantime [VcXsrv](https://sourceforge.net/projects/vcxsrv/) works quite well.


<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Does VcXsrv not give you the X you need?</p>&mdash; Social Justice Webserver (@joedevivo) <a href="https://twitter.com/joedevivo/status/997958930570493952?ref_src=twsrc%5Etfw">May 19, 2018</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 

You can add it to auto-start too which helps! After running it the first time it'll ask you to save the config, do so and then run windows-r and enter `shell:startup` to open the folder for auto-start, drop the file in there and call it a day.

Once `X` is started you need to tell the WSL shell where your display is you can do that by adding the line `export DISPLAY=localhost:0` at the end of the `~/.profile` file.

With that all set you should have a running emacs, feel free to skip the the next section or keep reading here if you want to know a few more fancies.

I really like the [Fira Code](https://github.com/tonsky/FiraCode) font, fortunately it comes with ubuntu so you can install it by running `sudo install fonts-firacode`. To enable it in emacs you can follow [their tutorial](https://github.com/tonsky/FiraCode/wiki/Emacs-instructions). Here is the gotcha I talked about while the font works I've not yet managed to get the code points to work, but it's not that big of a deal to me.

A second fancy but not required thing is a Emacs desktop icon. I really wanted the option to just click it and make it start without having to go through bash and running a command or having a console window open. There is a simple trick for that. Create a VB script somewhere with the follow content:

```basic
Set oShell = CreateObject ("Wscript.Shell") 
Dim strArgs
strArgs = "bash -c ~/start-emacs"
oShell.Run strArgs, 0, false
```

and a shell script in your home directory called start-emacs:

```bash
#!/bin/sh

export DISPLAY=localhost:0
emacs
```

That's it you can then link that to the laptop and give it the nice [spacemacs icon](https://raw.githubusercontent.com/nashamri/spacemacs-logo/master/spacemacs.ico).

### Windows / The hardware

There are a few tweaks to the system I hade to make it more frinedly to my use (read MY use, so YMMV).

### Touchpad

The touchpad just isn't at par with Apples touchpad, honestly none is and I suspect if you ever used a MacBook's touchpad you easiely agree. But XPS is one of the better ones, probably the best on a PC I've ever used and there are a few tweaks that make it more user friendly.

1. Disable tap to click, it's just super annoying if you're used the Mac touchpad.
2. disable the right side as a right click, two fingers like on the Mac just work fine and feel more natura (to a mac user).
3. Set 3 finger gestures to "Switching Desktops and showing desktops".

### Keyboad

The XPS uses what dell calls a maglev keyboard, which sounds really cool, it also types quite well. It's a bit clicky, that's not for everyone but pressing the key is distinct and you don't get this "did I press it or not" feeling. Not good or bad but I noticed that compared to the Mac the alt and ctrl keys are placed different. While on the Mac FN and Command/apple/windows are on the outside and alt and ctrl are on the inside the Dell does it the oposite way around. It takes some time to get used to but I think after a few days the way it's on the Dell feels more natural and better reachable (I still press Windows-D instead of Alt-D way too often :P).

Now the downer. Dell has made the horrible deicsion to put the page up/down keys right about the arrow keys. I hate it. It's unnatural and trying to count the time I went a page up instead of left resulted in an integer overflow. Fortunately Kevin suggested a toll called [AutoHotkey](https://www.autohotkey.com/). A simple script like this:

```
PgUp::Left
PgDn::Right
```

sovles the problem. AHK translates it to a standalone executable that you can put into the auto-start folder mentioned above. Effectively this re-binds the page up/down keys to left/right so that no matter which you end up hitting you get the arrow keys.
