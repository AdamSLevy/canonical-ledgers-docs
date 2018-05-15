# Factom node set up
This guide explains how to set up a new factom node on testnet or mainnet from
an existing pre-configured AMI.

## AMI
The AMI will be named something like "factomd docker swarm on zfs" and will
include "testnet" if it's for the testnet. The AMI includes the following
configurations:

- A separate ZFS pool for docker that mounts to `/var/lib/docker`. This
  contains the proper docker volumes for `factom_database` and `factom_keys`
with a pre-downloaded factom database.
- Docker installed and enabled and the factomd and associated images
  downloaded.
- Telegraf installed and configured for monitoring docker, factomd, and zfs.

## Security Groups
Both the `factom-mainnet` and `factom-testnet` VPCs have nearly identical
security groups specifically for running factomd via docker swarm. The security
groups have minor differences in their rulesets for mainnet and testnet.

All factomd via docker swarm nodes require the following groups:
- `factomd-docker-swarm`
- `restricted-ssh`
- `write-influxdb`

Additionally, if the node is a guard node, then also add the `guard-nodes`
security group, which allows factomd to peer with and accept peers from any IP
address.

If the node is an authority node, then instead add the `authority-nodes` group,
which restricts factomd peers to our VPC subnets.

## Instance Tags
When launching a new instance please add the following tags in AWS:
```
Name = <FQDN> e.g. node1.us-west-2.factom-main.net
VPC Name = factom-mainnet-1, factom-mainnet-2, OR factom-testnet
Factom Role = Follower, Guard, OR Authority
```

## DNS/Hostname
New nodes should be given a new node number that is unique within its VPC. The
hostname shall follow this convention:
```
node<node number>.<AWS region>.factom-<main or test>.net
```
So if we create a new node on mainnet in the VPC `factom-mainnet-1` which is in
AWS region `us-west-1`, then the hostname shall be
```
nodeX.us-west-1.factom-main.net
```
where X is the next highest node number within that VPC.

Since testnet only exists in the AWS region `us-west-2` testnet hostnames
always follow the form `nodeX.us-west-2.factom-test.net`.

This shall be the fully qualified domain name (FQDN) for the host.

Each host shall have three DNS entries:
- An A record with the name `public.<FQDN>` pointing to the public IP address.
- An A record with the name `private.<FQDN>` pointing to the private IP address.
- A CNAME record with the name `<FQDN>` pointing to `public.<FQDN>`.

This allows us to easily change whether the FQDN resolves to a public or
private IP address by simply updating the CNAME record. If an application
always depends on the public address than it can specify the `public`
subdomain.

Here is an example set of DNS entries for `node1.us-west-2.factom-main.net`.
```
node1.us-west-2.factom-main.net.           CNAME public.node1.us-west-2.factom-main.net.
public.node1.us-west-2.factom-main.net.	   A     52.39.179.179
private.node1.us-west-2.factom-main.net.   A	 10.0.2.38
```

## Initial setup after launch

### 1. Set hostname
```
# hostnamectl set-hostname node2.us-west-2.factom-main.net
```

### 2. Edit `/etc/hosts`
Update `/etc/hosts` to specify this server's FQDN to point to its private IP
and include the `private.nodeX...` domain as well. Use `ip a` to double check
the private IP address. This ensures that use of the FQDN immediately resolves
to the private IP address.

`/etc/hosts`
```
# Static table lookup for hostnames.
# See hosts(5) for details.
127.0.0.1       localhost
::1             localhost
10.10.1.77      node3.us-west-2.factom-test.net private.node3.us-west-2.factom-test.net
```

### 3. Edit `/etc/telegraf/conf.d/global-tags.conf`
Update `/etc/telegraf/conf.d/global-tags.conf` with the proper AWS region, VPC
name, and Factom Role. Use all lower case for tag names and values. Remember
that you can determine the availability zone easily from the 3rd octet in the
private IP address: A is 0, b is 1, and c is 2. It should look like this:

`/etc/telegraf/conf.d/global-tags.conf`
```
[global_tags]
  aws_az  = "us-west-2b"
  aws_vpc = "factom-testnet"
  factom_role = "guard"
```

### 4. Join swarm
Each node in the swarm needs a unique NodeID which is generated when you join
the swarm. So the AMI is initially configured to not be joined with any swarm.

For testnet refer to [this](https://github.com/FactomProject/factomd-testnet-toolkit#join-the-docker-swarm).

For mainnet refer to [this](https://github.com/FactomProject/factomd-authority-toolkit#join-the-docker-swarm).

The command is specifically not documented here since it may change. Use the
above links as the official documentation.

### 5. Set up the factomd docker container
If you are running a guard node you don't need to do anything since the
container will already have been set up and run once with the correct CLI args
as part of the AMI.

If you are running a authority node or if you need to change the CLI args for
some other reason you need to remove the container and recreate it with the new
CLI args. Removing the container will not remove the underlying images or
volumes.

Remove the container:
```
# docker rm factomd
```

Recreate the container:

For the correct command line arguments refer to official docs for
[mainnet](https://github.com/FactomProject/factomd-authority-toolkit#from-the-docker-cli-recommended-and-better-tested)
or
[testnet](https://github.com/FactomProject/factomd-testnet-toolkit#from-the-docker-cli-recommended-and-better-tested).

Regardless of whether you are adding CLI args for factomd, always add the
argument `-p "9876:9876"` after the other `-p` arguments. This exposes the
prometheus port that telegraf uses to collect metrics from factomd.

If you are setting up an auth node you will want to add `-exclusive_in` to the
CLI args for factomd.

If you are setting up an auth node you will also need to copy the server
identity configuration to the `factomd.conf` file.

*All* nodes need an updated special peer list in the `factomd.conf` file. Add
the new node's private DNS record with the appropriate port number in a space
separated list to the correct line in all of the nodes' `factomd.conf` files.
For testnet this looks like:
```
CustomSpecialPeers    = "private.node1.us-west-2.factom-test.net:8110 private.node2.us-west-2.factom-test.net:8110 private.node3.us-west-2.factom-test.net:8110"
```
For mainnet use `MainSpecialPeers`.

### 6. Enable services
The AMI comes with `telegraf.service` and `docker-start@factomd.service`
disabled since they both need initial configuration before being started.

Now they can be enabled:
```
# systemctl enable telegraf docker-start@factomd
Created symlink /etc/systemd/system/multi-user.target.wants/telegraf.service → /usr/lib/systemd/system/telegraf.service.
Created symlink /etc/systemd/system/default.target.wants/docker-start@factomd.service → /usr/lib/systemd/system/docker-start@.service.
```
Don't start the services yet.

### 7. Final checks
The AMI image can easily become stale and mistakes are easy to make when
preparing it. Please check the following and correct anything if it is wrong.

- `/etc/factomd.conf` should be a symlink to
  `/var/lib/docker/volumes/factom_keys/_data/factomd.conf`
- `/var/lib/docker/volumes/factom_keys/_data/factomd.conf` shall have
  permissions `640` and be owned by root.
- `/var/lib/docker/volumes/factom_database/` and all subdirectories shall be
  owned by root.
- `/etc/factomd.conf` shall have an identity if it is an auth node and shall
  NOT have an identity if it is a guard or follower node. If it is intended as
a backup to an auth node it is OK for the identity to be commented out.
- The hostname and the `/etc/hosts` file need to be correct.
- The `/etc/telegraf/conf.d/global-tags.conf` have been updated.
- The special peers need to be updated.
- Verify the ZFS mountpoints for `/var/lib/docker` and children: `zfs list`.

### 8. Restart
Do not use the AWS console to reboot or else your dynamic public IP address may
change. Use the command line to reboot and then verify the following.

- You can login.
- The hostname is properly set and fully displayed in the bash prompt.
- `telegraf.service` is running without errors.
- `docker-start@factomd.service` is running without errors.
- The `factomd` dashboard indicates that it has connections to our special peers.
- The correct auth identity is used by factomd if it is an auth node.
- The new node shows up with the correct hostname on `chronograf.canonical-ledgers.com`.

### 9. Alerts
Once we have alerts configured in kapacitor, there may be some additional steps
to have them include new nodes.
