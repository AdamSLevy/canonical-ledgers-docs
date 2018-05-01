# ZFS Replication

When we spin up a new factom node in our network we need to copy the factomd
mainnet dataset to the new EC2 instance. Use this procedure to take and send a
snapshot to a new ec2 instance.

## Prerequisites

- node1 with a storage pool with a dataset containing an updated factom
  database stored in the dataset `zfact/mainnet/factomd`.
- node2 an empty initialized storage pool called `zfact` and factomd installed.
- SSH access from node1 to node2
- Sudo access on both nodes

## Steps

### 1. Take a snapshot.
On node1.

We need factomd to be stopped to take a clean snapshot.
```
root@node1# systemctl stop factomd
```

Take the snapshot and use the `date` command to label it.
```
root@node1# zfs snapshot zfact/mainnet/factomd@$(date -u '+%F-%T')
```

You can confirm that the snapshot exists.
```
user@node1$ zfs list -t snapshot
NAME                                        USED  AVAIL  REFER  MOUNTPOINT
zfact/mainnet/factomd@2018-04-30-23:36:54  3.05M      -  24.5G  -
```

Restart factomd now to minimize downtime.
```
root@node1# systemctl start factomd
```

### 2. Set up the receiving node.
Switch to node2.

Start by creating our intermediate dataset with no mountpoint. We do this as a
convention to indicate what network the database is for.
```
root@node2# zfs create -o mountpoint=none zfact/mainnet
```

We need to be root to run `zfs receive` and since envoking commands with sudo
over SSH is problematic, we will create a named pipe that our user can write
to.
```
user@node2$ mkfifo ~/zfs-pipe
```

Now become root and prepare to receive the snapshot. (This _is_ easier than
using `sudo zfs receive...` because you won't be prompted for your password
until you start the send and then you might forget.)
```
user@node2$ sudo su
root@node2# zfs receive zfact/mainnet/factomd < ~user/zfs-pipe
```

### 3. Send the snapshot.
Switch back to node1.

Send the transfer using `pv` to display progress. You will need to use your
user account that has sudo permissions and ssh access to node2.
```
root@node1$ sudo zfs send zfact/mainnet/factomd@2018-04-30-23:36:54 | pv | \
    ssh node2 'cat - > ./zfs-pipe'
```
Wait for the transfer to complete. For reference 29Gigs transferred across
availability zones in 4 minutes and 50 seconds.

### 4. Set the mount points symlinks on node2.
Switch back to node2.

Verify that the dataset has been created properly.
```
user@node2$ zfs list
NAME                    USED  AVAIL  REFER  MOUNTPOINT
zfact                  13.5G  17.3G    24K  /zfact
zfact/mainnet          13.5G  17.3G    24K  none
zfact/mainnet/factomd  13.5G  17.3G  13.5G  none
```

Set the proper mountpoint.
```
root@node2# zfs set mountpoint=/db/factomd zfact/mainnet/factomd
```

Set proper permissions.
```
root@node2# chown factomd:factomd -R /db/factomd
```

Set up symlink.
```
root@node2# rm -r /var/lib/factomd
root@node2# ln -s /db/factomd /var/lib/
```

Verify correct setup of symlinks within `/db/factomd`. Note that `.factom` is a
symlink to `./`.
```
user@node2$ ls -al /db/factomd
total 7
drwxr-xr-x 3 factomd factomd    4 Apr 30 15:56 .
drwxr-xr-x 3 root    root    4096 Apr 30 15:10 ..
lrwxrwxrwx 1 factomd factomd    2 Apr 30 15:56 .factom -> ./
drwxr-xr-x 3 factomd factomd    5 Apr 27 02:39 m2
```

### 5. Clean up snapshots.
On both node1 and node2.

Snapshots don't take up any space initially, but they can take up a lot of
space as the data changes and ZFS stores the differences. We don't need to keep
the snapshots around so we'll delete them from both hosts.
```
# zfs destroy zfact/mainnet/factomd@2018-04-30-23:36:54
```

