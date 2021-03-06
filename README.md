# Cluster-Storage

**Store, retrieve, share and persist data in clustered applications.**

One reason why we have to use Redis as cache storage is that we need to shared 
data between multiple processes, for example, in clustered applications. Redis 
is fast enough since it's based on memory, and reliable because it will 
occasionally flush data to file in order to backup.

However, the cluster itself could do the same thing. By using this package, an 
application can store data in the process memory, and sync them across the 
cluster. The data will be synced to file as well, and when a new process joining
in, the process can access the data as well.

## Example

For simplicity, this example uses the internal *cluster* module, however, 
**cluster-storage** doesn't rely on it and can work without it.

```javascript
import * as cluster from "cluster";
import * as os from "os";

if (cluster.isMaster) {
    let WORKER_ID = 1;

    for (let i=0; i < os.cpus().length; i++) {
        cluster.fork({ WORKER_ID: WORKER_ID++ });
    }
} else {
    // You could import cluster-storage at the top along with other modules, 
    // here I require it only in the worker process is just to emphasize that it
    // does not need cluster module for support.
    const { Storage } = require("cluster-storage");
    const { WORKER_ID } = process.env;

    function sleep(timeout) {
        return new Promise(resolve => setTimeout(resolve, timeout));
    }

    var store = new Storage("my-store");

    (async () => {
        if (WORKER_ID == "1") {
            // Only set data in worker 1.
            store.set("foo", "Hello, World!");
            store.set("bar", { name: "World", greeting: "Hello" });

            console.log(store.get("foo")); // Hello, World!
            console.log(store.get("bar")); // { name: "World", greeting: "Hello" }

            store.delete("foo");
            console.log(store.get("foo")); // null

            store.set("bar.name", "Mr. World");
            console.log(store.get("bar")); // { name: "Mr. World", greeting: "Hello" }

            store.set("bar.greeting", "Hi", 1000); // only exist for 1 sec
            console.log(store.get("bar.greeting")); // Hi

            await sleep(1000);
            console.log(store.get("bar.greeting")); // null
            console.log(store.get("bar")); // { name: "Mr. World" }
        } else {
            // Other workers just get those data. Only for example, due to IPC 
            // delay, you may not get expected result immediately here.
            console.log(store.get("foo")); // Hello, World!
            console.log(store.get("bar")); // { name: "World", greeting: "Hello" }
            console.log(store.get("foo")); // null
            console.log(store.get("bar")); // { name: "Mr. World", greeting: "Hello" }
            console.log(store.get("bar.greeting")); // Hi
            console.log(store.get("bar.greeting")); // null
            console.log(store.get("bar")); // { name: "Mr. World" }
        }
    });
}
```

## API

- `new Storage(name: string, options?: StoreOptions)`
    - `name` Every storage instance must have a unique name, the system uses
        it to distinguish and sync data between cluster processes.
    - `StoreOptions`
        - `path: string` A directory of where to store the file copy of data. 
            default value is `process.cwd()`.
        - `gcInterval: number` The interval time of garbage collection, default 
            value is `120000` (2 minutes).

- `store.filename: string` Returns the filename of where the data will be stored
    in the disk.
- `store.connect: boolean` Whether the storage channel has been connected.
- `store.closed: boolean` Whether the storage channel has been closed, once 
    closed, the data can no longer be manipulated.

- `set<T>(path: string, data: T, ttl?: number): T` Sets data to the given path. 
    This package uses [lodash](https://lodash.com) to manipulate data, if you're
    not familiar with the `path` lodash uses, check its document for more help.
- `get<T = any>(path: string): T` Gets data according to the given path, if no 
    data is found, `null` will be returned. Worth mentioned that this package 
    will only store the data that can be serialized with JSON.
- `has(path: string): boolean` Checks if data exists according to the given path.
- `delete(path: string): void` Deletes data according to the given path.

- `sync(): Promise<void>` This method is used to synchronizes data in a new 
    process, it will ask the manager process to flush existing data to the file 
    copy and then it will read the data from the file. In the manager process, 
    it will just read data and will not flush.

- `close(): Promise<void>` Closes the storage channel and flush out the data 
    copy. Once closed, the data can no longer be manipulated.
- `destroy(): Promise<void>` Clears the data entirely and delete the file copy,
    the storage channel will be closed after calling this method.

## Inside Details

This module stores data in memory locally, that's why you see the methods `set`,
`get` and `delete` are all synchronized, because they doesn't need any I/O 
operations. However, at the same time when calling `set` and `delete`, an IPC 
message will be broadcast to all cluster processes asynchronously for 
notification, and they will update the data copy in their own memories.

This package uses a module called 
[manager-process](https://github.com/hyurl/manager-process) to check if the 
current process is the manager process (a process that do management stuffs but 
not the master process, and is dynamic assigned), only the manager will flush 
data.

Due to many reasons, a worker process may exit unexpected (even the manager, 
fortunately the manager duty will be assigned to another worker immediately), 
when that happens, the system (e.g. PM2) may reboot a new worker to replace it,
in order to retrieve the lost data, the new process should call the `sync` 
method to synchronize data, so that keep the data always the same in all workers.

## Notice

The purpose of cluster-storage is for small applications to reduce the usage of 
third-party cache systems, but it's not designed for large data, that's is the 
case, try Redis instead.