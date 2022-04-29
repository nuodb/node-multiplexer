# multiplexer.js

To run this example, you will need to set up a NuoDB database on your local machine. Below I will run through the commands to set up the database from scratch on bare metal:

## Creating the database
```
nuocmd create database --db-name test --archive-path $NUODB_HOME/var/opt/test-archive --server-id nuoadmin-0
nuocmd create database --db-name test --dba-user dba --dba-password dba
```
The database only needs to be created once, after that it will persist and only needs to be started with `nuocmd start database --db-name test`

## Starting TEs

At this point, your domain should look like this:

```
ecoombes@ecoombes-VirtualBox:~/node-multiplexer/example$ nuocmd show domain
server version: 4.3-4-a942bb6efd, server license: Community
server time: 2022-04-29T12:25:29.991, client token: b78dd9f089cd9e819b1104b149585fadf3ac7338
Servers:
  [nuoadmin-0] ecoombes-VirtualBox:48005 [last_ack = 2.19] ACTIVE (LEADER, Leader=nuoadmin-0, log=13/3418/3418) Connected *
Databases:
  test [state = RUNNING]
    [SM] ecoombes-VirtualBox:48006 [start_id = 67] [server_id = nuoadmin-0] [pid = 6277] [node_id = 1] [last_ack =  1.54] MONITORED:RUNNING
```

The next step is to start multiple TEs with labels.

```
nuocmd start process --db-name test --engine-type TE --this-server --labels shard 0
nuocmd start process --db-name test --engine-type TE --this-server --labels shard 1
nuocmd start process --db-name test --engine-type TE --this-server --labels shard 2
```

When looking at the output of `nuocmd show domain`, you should now see 3 TEs running.

## Changing numShards in the Admin Key-Value Store

```
nuocmd set value --key numShards --value 1 --unconditional
```

Before execution, ensure that you have set the value of the `numShards` key in the admin KV store. During execution, you can change the value for the `numShards` key, and see that it affects the way that connections are directed among TEs.

## Reading the Output

When starting with a value of 1 for numShards, we will see the following output:
```
rows for pre-mapped id 0 -- should go to shard 0
[ { '[GETNODEID]': 2 } ]
rows for pre-mapped id 1 -- should go to shard 0
[ { '[GETNODEID]': 2 } ]
rows for pre-mapped id 2 -- should go to shard 0
[ { '[GETNODEID]': 2 } ]
```

The multiplexer is only aware of 1 shard, and so it redirect all connection requests to the TE labeled with shard 0.

However, if we change the value of `numShards` to 2 during execution, we will see the following output:

```
comissioning shard 1
shard change from 1 to 2!
rows for pre-mapped id 0 -- should go to shard 0
[ { '[GETNODEID]': 2 } ]
rows for pre-mapped id 1 -- should go to shard 1
[ { '[GETNODEID]': 3 } ]
rows for pre-mapped id 2 -- should go to shard 0
[ { '[GETNODEID]': 2 } ]
```

We can see that some requests are now being redirected to shard 1, as the multiplexer is made aware of the change in the number of shards. The `poll` function detected the change in the shards, commissioned a new shard, and altered the `shardMapper` to redirect requests to the new shard.

This also works in reverse, with the multiplexer being able to commission and decommission multiple shards at the same time based on a changing domain for the NuoDB database.

```
// numShards changed to 3
comissioning shard 2
shard change from 2 to 3!
rows for pre-mapped id 0 -- should go to shard 0
[ { '[GETNODEID]': 2 } ]
rows for pre-mapped id 1 -- should go to shard 1
[ { '[GETNODEID]': 3 } ]
rows for pre-mapped id 2 -- should go to shard 2
[ { '[GETNODEID]': 4 } ]
```

```
// numShards changed to 1
decomissioning shard 2
decomissioning shard 1
shard change from 3 to 1!
rows for pre-mapped id 0 -- should go to shard 0
[ { '[GETNODEID]': 2 } ]
rows for pre-mapped id 1 -- should go to shard 0
[ { '[GETNODEID]': 2 } ]
rows for pre-mapped id 2 -- should go to shard 0
[ { '[GETNODEID]': 2 } ]

```


