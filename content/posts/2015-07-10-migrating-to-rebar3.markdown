---
title: "migrating to rebar3"
date: 2015-07-10
tags: ["erlang"]
---

# A long journey from rebar2 to rebar3

Rebar 3 has recently started to surface out of alpha state and entered beta1, about time for the crazy people like me to abandon tried and tested tools to venture into the great vastness of the unknown!

So with a backpack, hiking shoes, food for about a week and a direct line to the `rebar3` IRC channel I set off to migrate [sniffle](https://github.com/project-fifo/sniffle) from `rebar2` to `rebar3`. Now, after it looks like everything is working, I want to write up what exactly went down.

The complete delta can be seen [here](https://github.com/project-fifo/sniffle/pull/2/files?diff=split) please be ware that the upgrade kicked of a bit of a chain reaction with updating libraries too.

# 3 =/= 2 + 1
The most important thing I found, or rather the biggest misconception I had, is that `rebar3` is the next iteration of `rebar2`. This lead to a lot of misery on my part. `rebar3` is an entirely different application, the workflow is different, the logic is different and the behavior is different. Just dropping it and expecting everything to keep working the same will not end well. Treat it like migrating to a different build tool and things are a lot easier.

# The simple stuff

## Folders and files

Some of the directories change, `deps` no longer exists and __has to be deleted__. It also can be removed form the `.gitignore` file. Instead `_build` now takes its place, somewhat, it's different but it can go into `.gitignore`.

The same way `ebin` doesn't exist any longer and __should be deleted__. The former `ebin` now lives also in `_build` so we don't need to add anything new to the `.gitignore` file here. The old `ebin` will take priority over the `.beam` files generated in `_build`. That said I was pointed to the fact that there are valid reasons to have it around, for example to prevent `rebar3` to generate the `.app` file from `.app.src` or if there are additional files compiled by another tool other then `rebar3`.

Now I said and even said it in bold that those folders __HAVE TO BE DELETED__ that is because they do, if not an axe murderer will come by your house and kill your cat, seriously, I was just lucky that I didn't have a cat so he left disappointed. Aside of the axe murderer, `rebar3` will also rather unexpectedly load things from there, which, unlike then the axe murderer, did affect me and cause quite some headache.

There is a new file, `rebar.lock` which you want to add to your repository, not ignore it, it will keep track of the versions of libraries that are into the `_build` directory and that should go there if they don't already exist.

## Commands

The `rebar get-deps` is deprecated so is `rebar update-deps`, you don't need them any more, `rebar3` figures out itself when dependencies need to be installed or updated (from the `rebar.lock` file). There is a new `rebar3 deps` command, which has nothing to do with the old commands, instead it is used to give a list of the dependencies of your project (but not the sub-dependencies).

`rebar doc` is now called `rebar3 edoc` that should be noted.

`rebar3 dialyzer` is new, it replaces the old workflow of running dialyzer on its own and does all the building and checking `.ptl` files for you. The old trick of `grep`-ing away known errors to mitigate them is not working due to the mixed output however I was told that erlang 18 comes with a [`-dializer` pre-compiler directive](http://www.erlang.org/doc/man/dialyzer.html) can be used to handle this. I am not really sure about this especially with third party libraries. 

The handling of dependencies changed too, `skip_deps=true` is no longer needed. Nor is `-r` if you are using a `apps/*/...` structure for your project. Along with those the `-D` flag is gone now, it can however the same can be achieved with profiles - later more to that.

`generate` was replaced by `release` and it now uses `relx` and not `reltool`. If you are one of the poor sods (like me) that was using `reltool` you are into a lot more fun here but that is mostly beyond the scope. If you used `relx` before this should be straight forward, just that the config now lives in the `rebar.config`. Existing `relx.config` files will still be used as long as no `relx` section exists. It should be noted that this also takes care of linking instead of copying files when used with `{dev_mode, true}` which can be very nice for developing. 

## Profiles

Now there is probably a lot to say, it is a way to handle differences in behavior, and can for example replace the `-D` flag like this: `{profiles, [{long, {erl_opts, [{d, LONGTESTS}]}`. I haven't fully grasped the power and best practice of this and there is a good [article in the docs](http://www.rebar3.org/docs/profiles) about this so I won't dive further into that.

Something worth pointing out before moving on is that __everything__ that can be in a `rebar.config` can be in a profile, including plugins, dependencies, erlang options and so on. This makes it incredibly powerful.

## Plugins

[Plugins](http://www.rebar3.org/docs/using-available-plugins) have changed a bit and become a lot more important. Some common tasks in `rebar2` now live in a plugin instead of being part of the core system. The most notable here is probably the Port Compiler (or `pc` as the plugin is called) which is used for building NIFs (like eleveldb).

Hex comes as a plugin, which is __really__ nice, however the plugin is needed to publish not to fetch dependencies. This plugin could happily go into the global config, yes there is a global config in `~/.config/rebar3/rebar.config`. However it is best to keep other plugins out there.

The EQC (QuickCheck) plugin is very nice if you have quick check, either the free or the commercial version. It should be pointed out here __not__ to put this in the global config, no matter how tempting it is or the axe murderer will come back. Other then that you can now put the properties into a `eqc` folder and separate them from tests and it is no longer needed to wrap them in `-ifdef(EQC)`  and `-ifdef(TEST)`. What is especially nice here is that it picks up on the same naming as [quickcheck-ci](http://quickcheck-ci.com/) so that will make things easier.

## Dependencies

This is a quite big topic, but it can be summed up in: forget everything you know about rebar's handling of dependencies it's invalid now.

Perhaps the most obvious change is that in addition to source dependencies you can now include hex packages. The packages can take the form: `dflow` as 'the latest version' (or the version fitting to other packages), or as `{dflow, "0.1.6"}` to pick a specific version (more details [here](http://elixir-lang.org/docs/stable/elixir/Version.html)).

Using packages has a huge advantage, they are cached locally which makes fetching them, especially for big projects, __a lot nicer__.

Now my experience with rebar2 was that dependencies were handled by just cloning all of the dependencies in the `deps` folder and then adding them to the library path. This also had the effect that order didn't really matter. For example you could happily include header files from projects you were not depending on in the application including it.

Now rebar3 is actually caring about what you do. For example I ran into the following situation. I have an application `sniffle` this application has file `include/sniffle_version.hrl`. Now `sniffle` was depending on `sniffle_watchdog`, however `sniffle_watchdig` was including `include/sniffle_version.hrl`


```
     +-------------+             +-------------+
     |   sniffle   |<------------|  watchdog   |
     +-------------+             +-------------+
            |                           ^
            |                           |
            |                           |
            |                           |
            |                           |
+-----------------------+               |
|  sniffle_version.hrl  |---------------+
+-----------------------+
```

This setup is no problem with `rebar2`, those files where in `apps/sniffle/include` and works great the file exist and that is all that's needed. __However__, with rebar3 this approach is problematic, since  `sniffle_watchdog` does not depend on `sniffle` it will not exist when `sniffle_watchdog` is compiled. This means that I needed to include `sniffle` in `sniffle_watchdog` which is not possible since it would create a circlular dependency. The solution for this was simply to put the version header in na own application that gets included into both.

```
+-------------+             +-------------+
|   sniffle   |<------------|  watchdog   |
+-------------+             +-------------+
       ^                           ^
       |                           |
       |                           |
       |                           |
       | +-----------------------+ |
       +-|    sniffle_version    |-+
         +-----------------------+
```


Another slightly related topic is that when building releases now the content of the app file matters more, that is probably my own shortcoming that I ran into the problem but I did not include many library applications into the `application` section of the `.app.src` file. That lead to them missing in the release and the application dying a horribly painful death. I found the following code snipped rather helpful to track what applications were missing, and then a lot of manual labour to find where they should be included.

```bash
ls -1 _build/default/rel/sniffle/lib/  | sed 's/-.*//g' | sort > rlibs
ls -1 _build/default/lib | sort > libs
vimdiff libs rlibs
```

## The bottom line

After working with it a bit I think rebar3, when treated as it's own tool and not a iteration of rebar, is going to be huge improvement over existing erlang build tools, both erlang.mk, rebar2, and most likely any of the others lurking in the shadows.

The devs are very friendly and responsive and have helped me a great deal during this rather interesting exercise and deserve a lot of credit for the work and for putting up with the involved hatred and anger they receive.

Yes rebar3 is a learning curve and in the beginning it can be quite steep, but so does any other tool to be fair. It still is in beta (for a good reason), but bugs are fixed very fast and the help debugging them is outstanding.

If you require a rock solid tool today it is probably best to wait a bit longer until the final release but that said I have come a full circle, from utter hatred and frustration (on day one) to loving it after a week and will be using it from now on.
