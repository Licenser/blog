---
title: "Getting started with riak_test and riak_core"
date: 2013-03-31
---

If you don't know what `riak_core` is, or don't have a `riak_core` based application you'll probably not take too much practical use of this posts you might want to start with [Ryan Zezeski's "working" blog try try try](https://github.com/rzezeski/try-try-try) and the [rebar plugin](https://github.com/basho/rebar_riak_core).

That said if you have a `riak_core` app this posts should get you started on how to test it with `riak_test`. We'll not go through the topic of how to write the tests itself, this might come in a later post also the `riak_kv` tests are a good point to start.

Please note that the approach described here is what I choose to do, it might not be best practice of the best way for you to do things. I also will link to [my fork](https://github.com/Licenser/riak_test) of `riak_test` instead of the [official one](https://github.com/basho/riak_test) since it includes some modifications required for testing apps other then `riak_kv` I hope those modifications will be merged back at some point but for now I want to get it ironed out a bit more before making a pull request.

## What is riak_test?
So before we start; a few words to `riak_test`. `riak_test` is a pretty nice framework for testing distributed applications, it is, just as much as all the other `riak_` stuff created by [Basho](http://basho.com), pretty darn awesome.

At it's current state it is very focused on testing `riak_kv` (or `riak` as in the database) but from a first glance a lot of functionality is very universal and after all `riak_kv` is also built on top of `riak_core`, so modifying it to run with other `riak_core` based apps is pretty easy.

## The setup

Since I will be testing multiple `riak_core` apps and not just one I decided to go the following path: Have the entire setup in a git repository, then have one branch for general fixes/changes to `riak_core` then have one branch for each application I want to test that is based on the general branch so common changes can easily be merged so it will look like this:


```
---riak_test--------- (bashos master tree)
   `---riak_core----- (modifications to make riak_test work with core apps)
    ` `  `---sniffle- (tests for sniffle)
     ` `---snarl----- (tests for snarl)
      `---howl------- (tests for howl)
```

We'll go over this and setting up tests for the [howl application](https://github.com/project-fifo/howl) it's rather small and simple and it's easier to follow along with something real instead of a made up situation.

## Getting started

Step one of getting started is to get a clone from the `riak_test` repository, that's pretty simple (alter the path if you decided to fork):

```
cd ~/Projects
git clone https://github.com/Licenser/riak_test.git
cd riak_test
```

Now we branch off to have a place to get our `howl` application but first we need to checkout the riak_core branch to make sure we get the changes included in it:

```
git branch howl && git checkout howl can be git checkout -b howl
```

Okay that's it for the basic setup not that bad so far is it?

## Configuration
Next thing we need to do is create a configuration, at this point we'll assume you don't have one yet so we'll start from scratch, if you add more then one application later on you can just add them to an existing configuration.

`riak_test` looks for the configuration file `~/.riak_test.config` and reads all the data from there so we'll first need to copy the sample config there:

```
cp riak_test.config.sample ~/.riak_test.config
```

Next step is to open it in your favorite editor, you'll recognize it's a good old Erlang config file with tuples to group sections. We'll be ignoring the `default` section for now, if you're interested in it the documentation is quite good!

So lets go down to where it reads:

```erlang
%% ===============================================================
%%  Project-specific configurations
%% ===============================================================
```

Here is where the fun starts, you'll see a tuple starting with `{rtdev,` - note this `rtdev` has nothing whatsoever to do with the `rtdev` that is in the `default` section as `{rt_harness, rtdev}`. The `rtdev` in the project part is just the name of the project, since your project is named `howl` not `rtdev` we'll go and change that first.

```erlang
{rtdev, [
```

Now we can go to set up some variables first up the project name and executables, the name itself is just for information (or if you use `giddyup`) the executables are how your application is started, since our application is named `howl` it's started with the command `howl` and the admin command for it is `howl-admin`.

```erlang
    %% The name of the project/product, used when fetching the test
    %% suite and reporting.
    {rt_project, "howl"},

    {rc_executable, "howl"},
    {rc_admin, "howl-admin"},
```

With that done come the services, those are the buggers you register in your _app.erl file, lets have a look at the howl_app.erl:

```erlang
%...
            ok = riak_core_node_watcher:service_up(howl, self()),
%...
```

So we only have one service here we need to watch out for, named, you might guess … right `howl` that makes the list rather short:


```erlang
    {rc_services, [howl]},
```

Now the cookie, it's a bit hidden in the code that you need to set it but you do, you will need this later so remember it! Since I am bad at remembering things I named it … `howl` … again.


```erlang
    {rt_cookie, howl},
```

Now comes the setup of paths, for this we've to decide where we want to put our data later on, I've put all my `riak_test` things in `~/rt/...` so we'll follow with this. Also note that my development process works on three branches:

* `test` - the most unstable branch.
* `dev` - here things go that should work.
* `master` - only full releases go in here.

This setup might not work for you at all, but since these are only path names it should be easy enough to adapt.

Note that by default `riak_test` will run tests on the `current` environment

```erlang
    %% Paths to the locations of various versions of the project. This
    %% is only valid for the `rtdev' harness.
    {rtdev_path, [
                  %% This is the root of the built `rtdev' repository,
                  %% used for manipulating the repo with git. All
                  %% versions should be inside this directory.
                  {root, "~/rt/howl"},

                  %% The path to the `current' version, which is used
                  %% exclusively except during upgrade tests.
                  {current, "~/rt/howl/howl-test"},

                  %% The path to the most immediately previous version
                  %% of the project, which is used when doing upgrade
                  %% tests.
                  {previous, "~/rt/howl/howl-dev"},

                  %% The path to the version before `previous', which
                  %% is used when doing upgrade tests.
                  {legacy, "~/rt/howl/howl-stable"}
                 ]}
]}
```

And that's it now the config is set up and should look like this:

```erlang
{rtdev, [
    %% The name of the project/product, used when fetching the test
    %% suite and reporting.
    {rt_project, "howl"},

    {rc_executable, "rhowl"},
    {rc_admin, "howl-admin"},
    {rc_services, [howl]},
    {rt_cookie, howl},
    %% Paths to the locations of various versions of the project. This
    %% is only valid for the `rtdev' harness.
    {rtdev_path, [
                  %% This is the root of the built `rtdev' repository,
                  %% used for manipulating the repo with git. All
                  %% versions should be inside this directory.
                  {root, "~/rt/howl"},

                  %% The path to the `current' version, which is used
                  %% exclusively except during upgrade tests.
                  {current, "~/rt/howl/howl-test"},

                  %% The path to the most immediately previous version
                  %% of the project, which is used when doing upgrade
                  %% tests.
                  {previous, "~/rt/howl/howl-dev"},

                  %% The path to the version before `previous', which
                  %% is used when doing upgrade tests.
                  {legacy, "~/rt/howl/howl-stable"}
                 ]}
]}
```

## Setting up the application

We've got `riak_test` ready to test now next we need to prepare howl to be tested, we'll only look at the `current` (aka `test`) setup since the steps for others are pretty much the same.

The first step is that we need the folder, so lets create it

```bash
mkdir -p ~/rt/raw/howl
cd ~/rt/raw/howl
```

Since howl lives with the octocat on github it's easy to fetch our application and checkout the test branch (remember `current` is on the test branch for me):


```bash
git clone https://github.com/project-fifo/howl.git howl-test
cd howl-test
git checkout test
```

And done, now since it's a `riak_core` app we should have a task called `stagedevrel` in our makefile which will basically generate three copies of `howl` for us in the folders `dev/dev{1,2,3}` and in the process take care of compiling and getting the dependencies. I prefer `stagedevrel` over the normal `devrel` since later on it will it easier to recompile code files (`make` is enough) because it links them to the right place instead of copying them.

```bash
make stagedevrel
```

Now we've got to do a bit of cheating, `riak_test` expects the root dir to be a git repository, that is why we can't just put the data in there directly, so we've to manually build the tree for riak core and set it up as a git repository.


```bash
mkdir -p ~/rt/howl
cd ~/rt/howl
git init

cat <<EOF > ~/rt/howl/.gitignore
*/dev/*/bin
*/dev/*/erts-*
*/dev/*/lib
*/dev/*/releases
*/dev/*/share
*/dev/*/snmp
EOF
git add .gitignore
```

Now we need to link our devrel files. Since howl uses [cuttlefish] I have to copy the `howl.config.example` file to `howl.config` into `etc` directory, this may be different for your setup. 

```bash
export RT_BASE=~/rt/howl/howl-test
export RC_BASE=~/rt/raw/howl/howl-test
for i in 1 2 3 4
do
  mkdir -p ${RT_BASE}/dev/dev${i}/
  cd ${RT_BASE}/dev/dev${i}/
  mkdir data etc
  touch data/.gitignore
  ln -s ${RC_BASE}/dev/dev${i}/{bin,erts-*,lib,releases,share,snmp} .
  cp ${RC_BASE}/dev/dev${i}/etc/howl.conf.example etc/howl.conf
done
  
```

We still need to edit the `vm.args` in `dev/dev{1,2,3,4}/etc/`  since we need to set the correct cookie - I hope you still remember yours, I told you you'd need it (if not you can just look in the `~/.riak_test.config`)!

That's it.

How add the content to the git repository you initiated and commit it.

```bash
cd $RT_BASE && git add . && git commit -am 'First build.'  
```

## Running a first test

In the `riak_core` branch of `riak_test` I've moved all the `riak_kv` specific tests from `tests` to `tests_riakkv` so you still can look at them but I left one of them in `tests`, namely the basic command test - it will check if your applications command (`howl` in our case) is well behaved.

We'll want to run it to see if `howl` is a good boy and does well to do so we'll need to get back into the `riak_test` folder and run the `riak_test` command:

```bash
cd ~/Projects/riak_test
./riak_test -t tests/* -c howl -v -b none
```


I'd like to explain this a bit, the arguments have the following meaning:

* `-t tests/*` - we'll be running all tests in the folder `tests/`.
* `-c howl` - our application we want to test is named howl, this is the first element of the tuple we put in our config file when you remember.
* `-v` - This just turns on verbose output.
* `-b none` - This is still a relict from the `riak_kv` roots, it means which backend to test with, since we don't have backends at all we'll just pass none which means riak_test will happily ignore it.

That's it! Now go and test all the things!


This is the first part of a series that goes on [here](/blog/2013/04/09/writing-your-first-riak-test-test-yes-i-know-there-are-two-tests-there/).
