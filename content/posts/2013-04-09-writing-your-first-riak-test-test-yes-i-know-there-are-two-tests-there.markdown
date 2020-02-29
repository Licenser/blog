---
title: "Writing your first riak test test (yes I know there are two tests there)"
date: 2013-04-09
---

As promised in a [previous post](/blog/2013/03/31/getting-started-with-riak-test-and-riak-core/) I'll talk a bit about writing tests for `riak_test`. To start with the obvious it's pretty simple and pretty awesome. `riak_test` gives you the tools you've dreamed of when testing distributed `riak_core` applications:

* a backchannel to communicate and execute commands on the nodes.
* a nice and way to bring up and tear down the test environment.
* helper functions to deal with the `riak_core` cluster.
* something called `intercepts` that allow you to mock certain behaviours in the cluster.
* all the power of Erlang.


## How a test works
Tests have a very simple structre they pretty much contain a single function: `confirm/0`

This function gets called when the test is executed and should return `pass` when everything works well or throw an exception when not. The actual test are simple unit asserts you use.

That in itself is not really overly exciting and those of you with a short attention span might start to thing 'boooooooring' so lets look at the exciting part.

## Starting your applicatio
`riak_test` offers a way to start instances of your application and communicate with them, the common pattern is to start one (or more) nodes as first part of the script and check of they are up and running. That could look something like this:

```erlang
confirm() ->
    [Node] = rt:deploy_nodes(1),
    ?assertEqual(ok, rt:wait_until_nodes_ready([Node])),
    pass.
```


This is a minimal test it sets up one instance of our application and waits for it to be ready.


`rt:deploy_nodes(1)` will deploy one node, the id of the node (a atom with that identifies it to erlang) will be sotred in `Node`, you can deploy more nodes by increasing the number in `rt:deploy_nodes/1`.

`?assertEqual(ok, rt:wait_until_nodes_ready([Node]))` will make the test wait for the nodes to be ready, ready here means that all ring services we defined in the config are provided.

The node will be running in its own Erlang VM and have the test suite connected as a hidden node. This is the first thing that is truly interesting, since the connection will allow us to run rpc calls on the node this is the first thing that is truly fun.

## An official channel
Now we've a basic test running have our nodes to be started up and the test waiting until all is started and happy.

So chances are that aside of this back channel communication the node provides some kind of API, and we want to be able to connect to this API to run our tests. Un my case it's a simple TCP port that announces itself over mdns, we could simply listen to the broadcast and use the information it provides to talk to the node. This would work, as long as we've a single node, the moment we've to we would never know which node we're talking to and that would make testing hard.

So backchannel to the rescue! We'll just get the nodes configuration from the host, for my application I store this kind of information in the application configuration so I've made a function that given a `Node` returns `IP` and `Port` for it to talk to plus one to send data:

```erlang
node_endpoing(Node) ->
    {ok, IP} = rpc:call(Node, application, get_env, [mdns_server_lib, ip]),
    {ok, Port} = rpc:call(Node, application, get_env, [mdns_server_lib, port]),
    {IP, Port}.

call(Node, Msg) ->
    {IP, Port} = node_endpoing(Node),
    lager:debug("~s:~p <- ~p", [IP, Port, Msg]),
    {ok, Socket} = gen_tcp:connect(IP, Port, [binary, {active,false}, {packet,4}], 100),
    ok = gen_tcp:send(Socket, term_to_binary(Msg)),
    {ok, Repl} = gen_tcp:recv(Socket, 0),
    {reply, Res} = binary_to_term(Repl),
    lager:debug("~s:~p -> ~p", [IP, Port, Res]),
    gen_tcp:close(Socket),
    Res.
```

There is some `gen_tcp` stuff in there but lets just ignore it for now it's detail not important, the first function is the more interesting one it uses the `rpc` module to execute the commands on the node we just started which is quite awesome.

As a note I've put those functions into `rt_<applicatin name>` so for example rt_sniffle.

## Testing the API

With `call/2` we've now a way to send data directly to the node over the official API channel so lets add to our test. So lets get back to our `confirm/0 ` and add some real tests:

```erlang
confirm() ->
    [Node] = rt:deploy_nodes(1),
    ?assertEqual(ok, rt:wait_until_nodes_ready([Node])),
    ?assertEqual({ok,[]},
                 rt_sniffle:call(Node, {vm, list})),
    ?assertEqual(ok,
                 rt_sniffle:call(Node, {vm, register, <<"vmid">>, <<"hypervisor">>})),
    ?assertEqual({ok,[<<"vmid">>]},
                 rt_sniffle:call(Node, {vm, list})),                 
    pass.
```


It's as easy as this, this test will

* List the registered VM's and expect none to be there.
* Create a new VM and expect a ok result.
* List the registered vm's and expect the one we just registered to be present.


And that's it for basic testing with `riak_test` I'll follow up on this with an article over intercepts since they add another cool feature to `riak_test`.