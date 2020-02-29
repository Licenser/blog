---
title: "Plugins with Erlang"
date: 2013-03-19
---

## Preamble

Lets start with this, Erlang releases are super useful they are one of the features I like most about Erlang - you get out an entirely self contained package you can deploy and forget, no library trouble, no wrong version of the VM no trouble at all.

BUT (this had to come didn't it) sometimes they are limiting and kind of inflexible, adding a tiny little feature means rolling out a new release, with automated builds that is not so bad but 'not so bad' isn't good either. And things get worst when there are different wishes.

A little glimpse into reality: I'm currently working a lot on [Project FiFo](http://project-fifo.net) and one of the issues I faced is that - surprisingly - not everyone wants things to work exactly as I do. Which was a real shock, how could anyone ever disagree with me? Well ... I got over it, really I did, still solving this issue by adding one code path for every preference and making it configureable didn't looked like a good solution.

Also recently we are thinking a lot about performance metrics, and there are like a gazillion of them and if you pick two random people I think they want three different sets of metrics. Ask again after 5 minutes and their opinions changed to 7 new metrics and certainly not the old ones!

## Plugins

The problem is a very old one, extending the software after it was shipped, possibly letting the community extend it beyond what was dreamed of in the beginning. The solution is pretty much one day younger then the problem: plugins. Meaning a way to load code into the system.

Sounds easy but it is a bit more complex, just having something load into the VM doesn't do much good when it does not get executed in the proper place - so sadly this comes with extra work for the developer to sprinkle their code with hooks and callbacks for the plugins.

With this I'd like to introduce [eplugin](https://github.com/Licenser/eplugin) it's a very simplistic library for exactly that task - introduce plugins in an Erlang release or application. It takes care of discovering and loading plugins, letting them register into certain calls, a little dependency management on startup and provides functions to call registered plugins. Erlang comes with great tools already to the whole thing sums up to under 400 LOC.

## Types of plugins

I feel it's kind of interesting to look at the different kind of plugins that exist and how to handle the cases with [eplugin](https://github.com/Licenser/eplugin) also a post entirely without code would look boring.

### informative plugins

Sometimes a plugin just want to know that something happened but don't care about the result. `eplugin` provides the `call` (and `apply`) functions for that. A logger is a good example for this so lets have a look:

```erlang
%%% plugin.conf
{syslog_plugin,
  [{syslog_plugin, [{'some:event', log}]}],
  []}.
  
%%% syslog_plugin.erl
-module(syslog_plugin).
-export([log/1]).

log(String) ->
  os:cmd("logger '" ++ String ++ "'").
  
%%% in your code
  %%... 
  eplugin:call('some:event', "logging this!"),
  %%... 
```

Thats pretty much it, provided you've started the `eplugin` application in your code and put the plugins in the right place this will just work. You could also use this to trigger side effects, like delete all files when an error occurs to remove traces of your failure.

### messing around plugins

This kind of plugins process some data and return a new version of this data, we have `fold` for this case. Fold since it internally uses fold to pass the data from one plugin to another. there are many applications for that one would be to replace all occurrences of 'not js' with 'node.js' to prevent freudian typos in your texts.

```erlang
%%% plugin.conf
{not_js,
  [{not_js, [{'text:check', replace}]}],
  []}.
  
%%% not_js.erl
-module(not_js).
-export([replace/1]).

replace(String) ->
  re:replace(String, "not js", "node.js", [global]).
%%% in your code
  %%... 
  String1 = eplugin:fold('text:check', "I'm writing a not js application!"),
  %%... 
```

`fold` and `call` are the most interesting and important kind of plugins, they cover most if not all of the possible use cases of plugins so there is a special case left which I found useful to have.

### checking plugins

Checking plugins are plugins which are supposed to decide if something is Ok or not, they are pretty much a case of `fold` that returns true or false (or actually whatever is not true). But `eplugin` solves this too, with the `test` function! An example here is authentication

```erlang
%%% plugin.conf
{get_out,
  [{get_out, [{'login:allowed', no_really_not}]}],
  []}.
  
%%% get_out.erl
-module(get_out).
-export([no_really_not/1]).

no_really_not(Login) ->
  {forbidden, ["Dear ", Login, " we don't want you here go away!"]}.
  
%%% in your code
  %%... 
  case eplugin:test('login:allowed', "Licenser") of
     true ->
     	%%% huzza!
     Error ->
     	io:format("~p~n", [Error])
  end
  %%... 
```