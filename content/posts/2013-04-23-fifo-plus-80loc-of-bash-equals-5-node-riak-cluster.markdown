---
title: "FiFo + 80LOC of bash = 5 node riak cluster"
date: 2013-04-23
---

## The reason
The question 'why would I want at least 5 nodes' comes up very often in the #riak IRC channel, there is a [good explanation](http://basho.com/why-your-riak-cluster-should-have-at-least-five-nodes/). But that's boring, no one likes reading manuals, we, as engineers, like to try things out (aka. break stuff).

Only downside with that is that we need to set things up before we can break them, or even worst need to un-break it later to try out different things (aka. break it in different ways). Admittedly setting up a riak instance is __easy__ but setting up 5 and connecting them then break them and do all again to break them again, erm… I mean try things out of cause, can get really tedious and I for once am too lazy to bother with that.

## The goal
Make setting our breakage, erm test, bed setup as simple as possible, and whipping up things and tearing them down trivial, ideally have one simple command like `./riak.sh setup` to do that for us and `./riak.sh delete` undo it all for us to get back to a clean state.

## The tools

To build anything we'll need some tools, hammer and nails will not do us much good here so we are going to pick:

* [Project FiFo](http://project-fifo.net) - my favourite virtualisation tool (I am biassed I wrote it), but it's very easy to set up and very powerful.
* [The FiFo Console Client](https://github.com/project-fifo/pyfi) - we want to script things, a UI isn't helpful.
* bash - the simplest possible scripting tool.
* curl - since riak offers a http fronted it's a wonderful way to check if the system is up.
* [jsontool](http://trentm.com/json/) - a nifty utility to traverse JSON documents.

With that we should be set and good to go.

## The steps

We'll have to perform multiple steps to build our wracking ground for riak lets look at them one by one:


### Preparing the environment

Before we can begin we've to set up a few things, I'll not go into detail how to set up FiFo, there is a [good manual] for that with only like 5 steps required. So lets start at some of the script's variables:

```bash
#/usr/bin/env bash
PACKAGE="small"
DATASET="base64-1.9.1"
NET="7df94bc3-6a9f-4c88-8f80-7a8f4086b79d"
```


* `smal` is the name of the package created in FiFo, I picked something with 512MB of memory since that should be enough for now.
* `base64-1.9.1` is the dataset it means things are running in a solaris zone this also can be installed from the FiFo UI.
* `7df94bc3-6a9f-4c88-8f80-7a8f4086b79d` is the UUID of the network you can find that with `fifo networks list`


```bash
schroedinger:fifopy heinz [master] $ fifo packages list
                                UUID Name       RAM        CPU cap    Quota
------------------------------------ ---------- ---------- ---------- ----------
5f9f6c41-d700-4b4f-80f1-7350a71ed2e6 small      512 MB     100%       10 GB
schroedinger:fifopy heinz [master] $ fifo networks list
                                UUID Name       Tag                  First            Last
------------------------------------ ---------- ---------- --------------- ---------------
7df94bc3-6a9f-4c88-8f80-7a8f4086b79d test       admin        192.168.0.210   192.168.0.220
schroedinger:fifopy heinz [master] $ fifo datasets list
                                UUID Name       Version Type  Description
------------------------------------ ---------- ------- ----- ----------
60ed3a3e-92c7-11e2-ba4a-9b6d5feaa0c4 base       1.9.1   zone  A SmartOS ...
```

### Creating a VM with riak installed
Creating a VM is rather simple we need a little JSON and pipe it to fifo with a cat. Please note the section reading `user-script` here we make the setup. Here is how it looks.

```bash
cat <<EOF | fifo vms create -p $PACKAGE -d $DATASET
{
  "alias": "riak1",
  "networks": {"net0": "$NET"},
  "metadata": {"user-script": "/opt/local/bin/sed -i.bak \\"s/pkgsrc/pkgsrc-eu-ams/\\" /opt/local/etc/pkgin/repositories.conf; /opt/local/bin/pkgin update; /opt/local/bin/pkgin -y install riak; export IP=\`ifconfig net0 | head -n 2 | tail -n 1 | awk '{print \$2}'\`; /opt/local/bin/sed -i.bak \\"s/127.0.0.1/\$IP/\\" /opt/local/etc/riak/app.config; /opt/local/bin/sed -i.bak \\"s/127.0.0.1/\$IP/\\" /opt/local/etc/riak/vm.args; svcadm enable epmd riak"}
}
EOF
```

To get a bit better look user script section and remove the escape things:

```bash
# We configure pkgin to use the european mirror you might not need to do that.
/opt/local/bin/sed -i.bak "s/pkgsrc/pkgsrc-eu-ams/" /opt/local/etc/pkgin/repositories.conf;
# We update the pkgin database and install riak
/opt/local/bin/pkgin update;
/opt/local/bin/pkgin -y install riak;
# We find out what IP our VM has from within the VM.
export IP=`ifconfig net0 | head -n 2 | tail -n 1 | awk '{print $2}'`;
# We update the app.config and vm.args to use the 'public' ip instead of the 127.0.0.1
/opt/local/bin/sed -i.bak "s/127.0.0.1/$IP/" /opt/local/etc/riak/app.config;
/opt/local/bin/sed -i.bak "s/127.0.0.1/$IP/" /opt/local/etc/riak/vm.args;
# Start epmd and riak
svcadm enable epmd riak
```

### Waiting for riak

Now that is the first zone set up next we'll want to wait for riak to properly start up. This is needed since the commands are asynchronous and installing the packages can be a tad slow. But we can just to curl the http interface to check for this, so it's rather simple:

```bash
# We'll ask fifo for the IP of our first zone.
IP1=`fifo vms get riak1 | json networks[0].ip`
# Print some info so waiting is not so boring
echo -n 'Waiting until riak is up and running on the primary node.'
# now we curl the http interface every second to see if things are good.
until curl http://${IP1}:8098 2>/dev/null >/dev/null
do
	sleep 1
	echo -n '.'
done
# and we're done!
echo " done."
```

### Setting up the remaining zones

We're not going to get into too much details with this since it is pretty much working the same as the first VM with the only difference that the user-script holds two more lines:

```bash
for i in 2 3 4 5
do
	cat <<EOF | fifo vms create -p $PACKAGE -d $DATASET
  {
    "alias": "riak${i}",
    "networks": {"net0": "$NET"},
    "metadata": {"user-script": "/opt/local/bin/sed -i.bak \\"s/pkgsrc/pkgsrc-eu-ams/\\" /opt/local/etc/pkgin/repositories.conf; /opt/local/bin/pkgin update; /opt/local/bin/pkgin -y install riak; export IP=\`ifconfig net0 | head -n 2 | tail -n 1 | awk '{print \$2}'\`; /opt/local/bin/sed -i.bak \\"s/127.0.0.1/\$IP/\\" /opt/local/etc/riak/app.config; /opt/local/bin/sed -i.bak \\"s/127.0.0.1/\$IP/\\" /opt/local/etc/riak/vm.args; svcadm enable epmd riak; sleep 10; /opt/local/bin/sudo -uriak /opt/local/sbin/riak-admin cluster join riak@${IP1}; /opt/local/bin/sudo -uriak /opt/local/sbin/riak-admin cluster plan; /opt/local/bin/sudo -uriak /opt/local/sbin/riak-admin cluster commit"}
  }
EOF
	IP=`fifo vms get riak$i | json networks[0].ip`
	echo -n "Waiting untill riak is up and running on the node $i."
	until curl http://${IP}:8098 2>/dev/null >/dev/null
	do
    	sleep 1
		echo -n '.'
	done
	echo " done."

done
```

The two new lines are joining the node to the existing riak node which is quite easy, we can use $IP1 we generated in the first step too:
```bash
/opt/local/bin/sudo -uriak /opt/local/sbin/riak-admin cluster join riak@${IP1}
/opt/local/bin/sudo -uriak /opt/local/sbin/riak-admin cluster plan
/opt/local/bin/sudo -uriak /opt/local/sbin/riak-admin cluster commit
```

This is run that up and you've a 5 node riak cluster, and it's quick at last if you're in the US and have a good connection to the package repository.

[Here](https://raw.github.com/project-fifo/pyfi/master/examples/riak.sh) is this all slapped together.