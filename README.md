# node-multiplexer
An optional middle-ware layer for the nuodb node.js driver. The multiplexer allows the user to have a set of pools and define how connection requests are mapped between pools to take advantage of the performance benefits of using multiple Transaction Engines with NuoDB.

## Installation

### Manual Install
```
 git clone https://github.com/nuodb/node-multiplexer.git
 cd node-multiplexer
 npm i
```

The package can be cloned from github and installed by using the commands listed above. After that it can be imported like any other package.

### As A Dependency

```
 ...
  "dependencies": {
    ...
    "node-multiplexer":"nuodb/node-multiplexer#main"
    ...
  }
 ...
```
The package can be installed automatically by npm through it's github repo. Put the above line in the dependencies field of your package.json file.

## Usage

### Configuration and Initialization
```
const ShardMultiplexer = require('node-multiplexer')

const connectionConfig = {
  database: <your database name>,
  hostname: <admin host>,
  port: <admin port>,
  user: <db user>,
  password: <password>
}

const evenShardConfig = {
  id: 0,
  poolConfig: {
    id: 0,
    connectionConfig: {
      ...connectionConfig,
      LBQuery: `round_robin(first(label(shard 0)))`
    }
  }
}

const oddShardConfig = {
  id: 1,
  poolConfig: {
    id: 1,
    connectionConfig: {
      ...connectionConfig,
      LBQuery: `round_robin(first(label(shard 1)))`
    }
  }
}

const multiplexerConfig = {
  shards: [evenShardConfig,oddShardConfig],
  shardMapper: (id) => id % 2,
}

const multiplexer = new ShardMultiplexer(multiplexerConfig)
await multiplexer.init()
```

**shards**: Required. This is a list of config objects for shards to spin up on initialization. The shard config requires an `id` which may be a string or a number, and a `poolConfig` which is a config object for a nuodb connection pool. For more information about configuration objects for the nuodb connection pool, please visit the [node.js nuodb driver](https://github.com/nuodb/node-nuodb).

**shardMapper**: Required. The `shardMapper` is a function can take in any number of arguments, and should return either a string or a number. The return value of the shard mapper should correspond to the shard id. This function is invoked when making a call to the `requestConnection` method of the multiplexer, and the return value is used to select a shard to request a connection from to return to the user.

**poll**: Optional. A function which will be executed at an interval. The only argument passed to the `poll` function is the "this" argument, denoting the multiplexer instance executing the poll. The latest value returned from the `poll` function can be read by accessing the `pollResult` attribute of the multiplexer object.

**pollInterval**: Optional. An integer specifying the microseconds between execution of the poll function.

### Requesting and Releasing Connections
```
const connEven = await multiplexer.requestConnection(42)
const connOdd = await multiplexer.requestConnection(13) 
```
Building on the example above, the argument to the `requestConnection` method is determined by the arguments expected by the `shardMappper` function. The return value of the `shardMapper` function determines which shard/pool to retrieve a connection from. In this example when calling `requestConnection` with the argument `42`, the `shardMapper` function is called and returns a value of `0`. This return value will be used to pull a connection from the even shard, which has an id of 0. Likewise, when calling `requestConnection` with the argument `13` the `shardMapper` returns a value of `1`, and pulls a connection from the odd shard which has an id of 1.

```
await multiplexer.releaseConnection(connEven)
await multiplexer.releaseConnection(connOdd)
```
Building further on the example above, the method `releaseConnection` is used to return a connection back to it's proper pool.

### Commissioning and Decommissioning Shards

```
await multiplexer.decommissionShard(0)
await multiplexer.commissionShard(evenShardConfig)
```

Shards can be commissioned and decommissioned at will, as long as they have unique `id`s. Here we are decomissioning the even shard via it's `id` of `0`, and then commissioning it again using the same shard config.

### Advanced Usage

It is possible to take advantage of the closures functionality of node.js to dynamically commission and decommission shards, and change the `shardMapper` function from inside of the `poll` function. In this way, it is possible to react to a changing domain to leverage the full capabilities of NuoDB by increasing the likelihood of getting a cache hit by redirecting traffic requesting similar data to the same set of TEs. For a more detailed example demonstrating this behavior, please refer to the examples folder.





