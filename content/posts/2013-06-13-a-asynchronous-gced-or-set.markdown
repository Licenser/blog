---
title: "A asynchronously GCed OR Set"
date: 2013-06-13
---

Following the [article about Asynchronous garbage collection with CRDTs](/blog/2013/06/11/asyncronous-garbage-collection-with-crdts/) I experimented with [implementing the concept](https://github.com/Licenser/ecrdt/blob/master/src/vorsetg.erl). The OR Set is a very nice data structure for this  since it's rather simple and so is it's garbage!

To garbage collect the [OR Set](https://github.com/Licenser/ecrdt/blob/master/src/vorset.erl) we do the following, we take some of the elements of the remove set, and delete them from both the add and the remove set - this way we save the space for them and generate a new baseline.

First step was to implement the data structure described to hold the collectable items, I call it a [ROT](https://github.com/Licenser/ecrdt/blob/master/src/rot.erl) (Roughly Ordered Tree) it's a nice name for garbage related stuff ;) and it is treeish and mostly ordered.

The interface of the ROT is rather simple, Elements must be time tagged, in the form {Time, Element}. Where time must not be a clock, as long as the Erlang comparison operations work on it to give an order. Then it allows asking for full buckets, and removing buckets based on their hash value and newest message timestamp.

While the elements in a the OR set area already tagged with a timestamp, this timestamp records addition, not deletion so it would be misleading to use them since the ROT would think the remove happened when actually the addition happened and this would violate the rule that no event can travel back behind T<sub>100</sub>. As a result we'll have to double timestamp the removes - as in add a second when when it was removed.

So since the ROT has a very similar interface to a G Set (which implemented the remove set before) the change is trivial. **Remove**, **GC** and the **merge** function are more interesting.

### remove
```erlang
remove(Id, Element, ORSet = #vorsetg{removes = Removes}) ->
    CurrentExisting = [Elem || Elem = {_, E1} <- raw_value(ORSet),
                               E1 =:= Element],
    Removes1 = lists:foldl(fun(R, Rs) ->
                                   rot:add({Id, R}, Rs)
                           end, Removes, CurrentExisting),
    ORSet#vorsetg{removes = Removes1}.

```

`Id` defaults to a the current time in nanoseconds since it's precise enough for most cases, but can be given any value that provides timed order. Line 2 and 3 collect all observed and not yet removed instances of the element to delete, we then fold over those instances and add each of them to the ROT.

### GC
```erlang
gc(HashID,
   #vorsetg{
      adds = Adds,
      removes = Removes,
      gced = GCed}) ->
    {Values, Removes1} = rot:remove(HashID, Removes),
    Values1 = [V || {_, V} <- Values],
    Values2 = ordsets:from_list(Values1),
    #vorsetg{adds = ordsets:subtract(Adds, Values2),
             gced = ordsets:add_element(HashID, GCed),
             removes = Removes1}.
```

To GC the set we take the `HashID`, this is what the rot returns when it reports full buckets, and in line 6 remove it from the ROT. Thankfully the ROT will return the content of the deleted bucket, this comes in very handy, since in the process of garbage collecting the bucket we also need to remove the items once and for all from the add list as seen in line 9. We then record the GC action in line 10 to make sure it will applied during a merge.

Please note that currently this set, even so it is garbage collected still grows without bounds since the GC actions themselves are not (yet) garbage collected, this will be added in a later iteration.

### merge
```erlang
merge(ROTA = #vorsetg{gced = GCedA},
      ROTB = #vorsetg{gced = GCedB}) ->
    #vorsetg{
       adds = AddsA,
       gced = GCed,
       removes = RemovesA}
        = lists:foldl(fun gc/2, ROTA, GCedB),
    #vorsetg{
       adds = AddsB,
       removes = RemovesB}
        = lists:foldl(fun gc/2, ROTB, GCedA),
    ROT1 = rot:merge(RemovesA, RemovesB),
    #vorsetg{adds = ordsets:union(AddsA, AddsB),
             gced = GCed,
             removes = ROT1}.
```

Merging gets a bit more complicated due to the fact that we now have to take into account that values might be garbage collected in one set but not in the other. While merging them would do no harm it would recreate the garbage which isn't too nice. So what we do is applying the recorded GC actions to both sets first as seen in line 3 to 11 and then merge the remove values (line 12) finally the add values (line 13).

## Results
I set up some proper tests for the implementation, comparing the GCed OR Set (bucket size 10) with a normal OR Set, running 1000 iterations with a set of 1000 instructions composed of 70% adds and removes, 20% merges and 10% GC events. T<sub>100</sub> is a sliding time from the allowed collection of events older then the last merge.

Each stored element had the size of between 500 and 600 bytes (so there were 100 possible elements). A remove will always remove the stalest element, since they are added in random order this equals a random remove.

The operations are carried out of replicas copies of the set where add, and remove have a equal chance to be either happening just on copy A, or just on copy B, or on both replicas at the some time. GC operations are always carried out on both replicas but it should be noted that the GC operation does not include a merge operation so can be considered asynchronous.

All operations but the GC operation are executed exactly the same on the GCed OR set and the not GCed or Set in the same order and same spread.

At the end a final merge was performed and the resulting values compared for each iteration, no additional GC action takes place at the end.

Measured were both the space reduction per GC run and the final difference of size. Per GC run about 15% space was reclaimed and at the end the GCed set had a total space consumption of around 26% of the normal OR Set in average, 6% in the best and 143% in the worst case.

```
src/vorsetg.erl:389:<0.135.0>: [Size] Cnt: 1000,   Avg: 0.261,  Min: 0.062, Max: 1.507
src/vorsetg.erl:389:<0.135.0>: [ RS ] Cnt: 49221,  Avg: 0.866,  Min: 0.064, Max: 1.0
src/vorsetg.erl:389:<0.135.0>: [ GC ] Cnt: 49221,  Avg: 55.870, Min: 0,     Max: 6483
src/vorsetg.erl:389:<0.135.0>: [ MG ] Cnt: 98357,  Avg: 58.110, Min: 0,     Max: 6596
src/vorsetg.erl:389:<0.135.0>: [ OP ] Cnt: 344708, Avg: 38.539, Min: 0,     Max: 6916```
```

The numbers are from a test run, for readability truncated manually after 3 digest and aligned to be nicer readable. **Size** is total size at the end of the iteration, **RS** is the space reduction per GC run. **GC**, **MG** and **OP** are the time used for garbage collection, merging and other operations respectively, the numbers are per execution and measured microseconds. Time measurements also include noise that from additional operations required for the test and should not be seen as a useful benchmark!

## Conclusion
The GC method described seems to work, and not even too badly, in the course of experimenting with values it showed that the conserved space is heavily dependant on the environment like the bucket size chosen, the size of the elements, the add/remove ratio and the ratio on which merges happen.

The OR Set it was compared with was not optimised at all, but thanks to it's simplicity a rather good candidate, the gains on already optimised sets will likely be lower. (run with a [optimised OR Set](https://github.com/Licenser/ecrdt/blob/master/src/vorset2.erl) gave only 1 54% reduction in space instead of a 74% one with a normal OR Set).

The downside is that garbage collection takes time, so does merging, so a structure like this is over all slower then a not garbage collected version