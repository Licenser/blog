---
title: "Erlang and DTrace"
date: 2013-01-25
---
As part of [Project FiFo](http://project-fifo.net) I've invested some time to research into DTrace and Erlang, not the probes that are there since some time but a DTrace consumer - letting you execute DTrace scripts from within Erlang and read the results.

The result of this research is the [erltrace](https://www.github.com/project-fifo/erltrace) a consumer for DTrace implemented as a Erlang NIF. The NIF is based on the [node.js](https://github.com/bcantrill/node-libdtrace) and and [python](http://tmetsch.github.com/python-dtrace/) implementations - many thanks to them!

It's pretty easy to consume dtrace data from within erlang with [erltrace](https://www.github.com/project-fifo/erltrace) and I figured lets make a little demo. Since I'm not a big fan of reinventing the wheel and a simple demo had been done before, namely [heat-tracer](http://howtonode.org/heat-tracer).

It's a nice idea and simple enough to implement and cowboy gives a very nice base for that in Erlang. So there you go [dowboy](https://github.com/Licenser/dowboy). The HTML/JS part of the page is pretty much the same as in the original save for replacing socket.io with simple websockets, I'll skip this and the the cowboy parts to look directly in the interesting parts.


When connecting we set up a timer to inform us every second to gather the results from DTrace:

```erlang
websocket_init(_Any, Req, []) ->
	timer:send_interval(1000, tick),
	Req2 = cowboy_http_req:compact(Req),
	{ok, Req2, undefined, hibernate}.
```

Next up we handle incoming Websockets messages, this deals very simple. Every string that is send is considered a new dtrace script to execute.

The first part creates a new handler, this is pretty much a reference to the libdtrace internal data structures. Since we allow multiple scripts to be send we also have to ensure that the old sockets are closed before new ones are opened.

```erlang
websocket_handle({text, Msg}, Req, State) ->
    %% We create a new handler
    {ok, Handle} = case State of
                       undefined ->
                           erltrace:open();
                       {Old} ->
                           %% But we want to make sure that any old one is closed first.
                           erltrace:close(Old),
                           erltrace:open()
                   end,
```

Next we compile the dtrace script, [erltrace](https://www.github.com/project-fifo/erltrace) only takes lists as strings and not binaries so we've to convert it first and then we pass it along with our handle to get the script compiled. It will return `ok` if everything went well.

```erlang
    %% We've to confert cowboys binary to a list.
    Msg1 = binary_to_list(Msg),
    ok = erltrace:compile(Handle, Msg1),
```

Okay now that we've compiled the script we just need to tell dtrace to start running it, this happens with the `erltrace:go` call. Again it will return `ok` when everything is fine. Finally we just output the script as debug info and return.

```erlang
    ok = erltrace:go(Handle),
    io:format("SCRIPT> ~s~n", [Msg]),
	{ok, Req, {Msg1, Handle}};
```

Next up: reading the dtrace data, `erltrace:walk` do that for you and hand back a datastructur to parse. It is returned as `{ok, Data}` when there is something to handle.

```erlang
websocket_info(tick, Req, {Msg, Handle} = State) ->
     case erltrace:walk(Handle) of
         {ok, R} ->
```

Now since we have JSON on the other side we need to transform the data here Erlangs list comprehensions come to the rescue. Data is returned as `[{lquantize, [Name], { {BucketStart, BucketEnd
}, BucketCount}}]` and we want it in the form `{Name, [[BucketStart, BucketEnd], BucketCount]` so here you go. We can tehn simpley encode this with [jsx](https://github.com/talentdeficit/jsx) and send it over the wire:

```erlang
             JSON = [{list_to_binary(Call),[ [[S, E], V]|| { {S, E}, V} <- Vs]}|| {lquantize, [Call], Vs} <- R],
             {reply, {text, jsx:encode(JSON)}, Req, State, hibernate};
```

`ok` will be returned if there is no data yet to consume, we simply do nothing here.

```erlang
         ok ->
             {ok, Req, {Msg, Handle1}};
```


The last case is just for making things proper, if an error is returned we close stop the current handle and create a new one the same we did in the init.

```erlang
         Other ->
             io:format("Error: ~p", [E]),
             try
                 erltrace:stop(Handle)
             catch
                 _:_ ->
                     ok
             end,
             {ok, Handle1} = erltrace:open(),
             erltrace:compile(Handle1, Msg),
             erltrace:go(Handle),
             {ok, Req, {Msg, Handle1}}
     end;
```

Now that all put together and run it on a dtrace capable machine we get a nice little heatmap that updates every second:

![Heatmap](/images/posts/2013-01-25-erlang-and-dtrace-heatmap.png)

Neat isn't it?
