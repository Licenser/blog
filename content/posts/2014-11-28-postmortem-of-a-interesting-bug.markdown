---
title: "Postmortem of a interesting bug"
date: 2014-11-28
tags: ["fifo"]
---
## Symptoms
After a full network outage in a larger system (7 FiFo instances and, a few dozen of hypervisors, VM's in the 3 digit number) a small percentage of the VM's stored in FiFo lost information which package was assigned to them and which organization they belong to.

## Technical background
As part of planned maintenance on the switching layer the entire network was taken down. Effectively cutting the communication between any two systems in the network.  During the maintenance the FiFo system was kept "hot", no services disabled or paused. At the end of the maintenance window the network was as planned enabled again, reinstating communications.

## FiFo background
We will focus in the sniffle and chunter components since those are the relevant parts of sniffle that handle information related to the symptoms.
¯
Generally all fifo data is stored in CRDT's which provides conflict resolution and nearly loss free merging even of entirely divergent data.

### Sniffle
Sniffle is a distributed system that runs on top of riak_core, all (7) nodes are connected as a mash network via distributed erlang. While the nodes are connected data and functionality is split out in dynamo fashion, at a default N value of 3 that means every node handles (total data)*3/7 of the data.

In the case of a node failure the adjacent node takes over the function of the failed node. Missing data is resolved by a method called "read repair", meaning that when data is requested and 1 out of the 3 nodes responds with no data that "missing" data is repaired.

Once the node failure is repaired the system that took over the work for the failed node performs what is called a handoff, sending it's (supposedly) updated data to the returning node to bring it up to speed.


### Chunter
Under normal conditions chunter will send incremental updates about changing vm state to sniffle. If a chunter system for the first time connects to sniffle it will register it's VM's, meaning they are created if not existent, and assigned to the hypervisor. The registration contains nearly all relevant VM information with some exceptions that do not concern the hypervisor. The missing information was amongst the not updated information.

Sniffle nodes are discovered via mDNS broadcasts, the first broadcast received will trigger the initial registration sending the request to this node.


## What happened?
It is a combination of conditions that lead to the problem and explains why  only a small percentage of VM's were affected.

### During the outage
1) The network outage cut the communication between **all** sniffle nodes, so each node was on its own handling the full load of the requests.
2) The network outage cut communication of **nearly all** chunter nodes, only those who resided on a hypervisor that also contained a fifo zone did not loose connectivity.

### After the outage
Now there happened a race condition, on one side the distributed erlang started to re-discover the other nodes, reestablishing communication in sniffle and initializing handoffs (handoffs of nearly all parts).

On the other side the periodic mDNS broadcasts from the sniffle nodes triggered re-registration of the chunter nodes.

### The bug
The bug was triggered by the fact that during receiving a handoff a vnode took the wrong assumption that the received data must be newer then its own and overwrite the currently stored data instead of merging it.

Due to the usual repair during reads this turned out to be quite an edge case that was only triggered when:

1) The mDNS was received from a node that neither holds the right partition, neither is connected to a node holding the right partition.
2) the registration happened to three vnodes that did not hold that data since as long as one node held the data it would have been merged with this instead of re-created
3) The handoff on all three systems was performed before any read happened, since the read-repair would haver otherwise merged the data.
4) The handoff off all three systems was performed before AAE triggered, since AAE triggers a read-repair on inconsistent data.

## The fix
This was fixed rather simple, instead of overwriting existing data on a handoff, the handoff is now treated as a read-repair and old and existing data is merged.
