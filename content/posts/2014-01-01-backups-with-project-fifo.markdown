---
title: "Backups with Project FiFo"
date: 2014-01-01
---
With 0.4.3 FiFo introduces support for [LeoFS](http://www.leofs.org) and this allows for some quite nice new features. Most importantly it decouples FiFo's operations from storing big amounts of data which makes maintaining either of this much more sensible and scaling storage much more easy.

Then again while nice that is not the important part, just storing datasets somewhere else does not make much of a difference for most users but what LoeFS allows FiFo to store much more data then would be good in the old setup. 'A lot more' here means pretty much as much as you can store.

So with this options 0.4.3 FiFo introduce backups! Backups complement the snapshots already in the system for quiet a while but while snapshots were made to stay on the hypervisor backups are supposed to be shipped off to LeoFS. This not only helps to keep the number of snapshots limited, does not count against the local quota but also widens the failure domain.

And to make it better there is a sensible concept about incremental and full backups, that said there are a few limitations to be aware of:

* Backups that stay on the hypervisor will count against the local quota.
* Once a backup is moved away from the hypervisor it can't be restored without overwriting the current state.
* Restoring a backup might mean first deleting the local zfs volume for the vm.

But that aside that there are some very interesting things:

* While it's not possible to keep multiple branches with snapshots this is very well possible with backups.
* It's possible to make a difference between incremental and full backups choosing between recovery speed or space efficiency.


All that sums up to something quite awesome, it allows for proper grandfather backups concepts for VM's and they can even be scripted using the fifo python client. So here is an example how this could be done.

Lets quickly describe what we want to achieve:

* Every month we want a full backup.
* Every week we want an incremental backup
	* for the first week in a month towards the monthly full backup
	* for other weeks to the previous week
* Every day we want a incremental backup
	* for the first day of a week from the week backup
	* for other days from the previous day

To allow for the incremental backups we need to keep some of the backups around:

* monthly until the first weekly was done.
* weekly until the next weekly was done but not longer the the next monthly.
* daily until the next daily but not longer then the next weekly or monthly.

The FiFo backup code fortunately helps a lot with this, a important part of the logic is *'create a incremental snapshot and delete its parent from the hypervisor'* and that is exactly the behavior of the backup code when passing both a parent and requesting a delete.

Here some [code](https://github.com/project-fifo/pyfi/blob/master/examples/backup.sh) (with some comments added):

```bash
#!/usr/bin/env bash
fifo=fifo
vm="$2"
case $1 in
    monthly)
        $fifo vms backups $vm create monthly
        # After createing a new monthly snapshot we first delete the last weekly and daily backup
        last_daily=$($fifo vms backups $vm list -pH --fmt uuid,local,comment | grep 'daily' | grep 'YES' | tail -1)
        if [ ! -z "$last_daily" ]
        then
            daily_uuid=$(echo $last_daily | cut -d: -f1)
            # the -l flag tells FiFo to only remove the backup from the hypervisor.
            $fifo vms backups $vm delete -l $daily_uuid
        fi
        last_weekly=$($fifo vms backups $vm list -pH --fmt uuid,local,comment | grep 'weekly' | grep 'YES' | tail -1)
        if [ ! -z "$last_weekly" ]
        then
            weekly_uuid=$(echo $last_weekly | cut -d: -f1)
            $fifo vms backups $vm delete -l $weekly_uuid
        fi
        ;;
    weekly)
        last_backup=$($fifo vms backups $vm list -pH --fmt uuid,local,comment | grep 'monthly\|weekly' | grep 'YES' | tail -1)
        uuid=$(echo $last_backup | cut -d: -f1)
        $fifo vms backups $vm create --parent $uuid -d weekly
        # After creating a new weekly we need to make sure to delete the last daily one
        last_daily=$($fifo vms backups $vm list -pH --fmt uuid,local,comment | grep 'daily' | grep 'YES' | tail -1)
        if [ ! -z "$last_daily" ]
        then
            daily_uuid=$(echo $last_daily | cut -d: -f1)
            $fifo vms backups $vm delete -l $daily_uuid
        fi
        ;;
    daily)
        last_backup=$($fifo vms backups $vm list -pH --fmt uuid,local,comment | grep 'daily\|weekly' | grep 'YES' | tail -1)
        uuid=$(echo $last_backup | cut -d: -f1)
        type=$(echo $last_backup | cut -d: -f2)
        case $type in
            weekly)
                $fifo vms backups $vm create --parent $uuid daily
                ;;
            daily)
                $fifo vms backups $vm create --parent $uuid -d daily
                ;;
        esac
        ;;
esac

```
