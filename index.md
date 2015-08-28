Shardcache
==========

Shardcache is a distributed cache and storage system, originally inspired by
[groupcache](https://github.com/golang/groupcache), intended as a replacement
for [memcached](http://memcached.org). Note that shardcache does not include
code from those projects, and also the protocol for node and communication
is different.

Key Features
------------

The project is divided into a `libshardcache` library, implementing both
a client and a node, and a `shardcached` daemon, which implements a shardcache
node and an optional HTTP frontend to the cache.

### libshardcache

* Cache filling algorithm based on adaptive replacement that minimizes access
  to the underlying storage

* Automatically connects to its own peers and handles intra-node communication

* Ensures the items are fetched from the peers or from the storage only once
  even when multiple concurrent requests are accessing the same uncached item

* Supports SET operations. If the node which receives the SET operation is
  responsible for the specified KEY, the new value will be provided to the
  underlying storage (and to next GET requests). If the receiving node is not
  the responsible for the key, the request will be forwarded (through the
  internal communication channel) to the responsible peer which will eventually
  store the new value and make it available to all the shardcache nodes.

* Supports DEL operations. If the node which receives the DEL operation is
  responsible for the specified KEY, the key will be removed from the underlying
  storage and from the cache. If evict-on-delete is turned on (it is by default)
  an evict command will be sent to all other peers to force eviction from their
  cache. If the receiving node is not responsible for the key, it will still be
  removed from the local cache (if present) and the request will be forwarded
  through the internal communication channel to the responsible peer which will
  eventually remove the key from its storage and from the cache.

* Supports EVICT operations. Evicts differ from deletes in the sense that the key
  is only unloaded from the cache but not removed from the storage. Forcing
  evictions might be very useful to force new values to be visible as soon as
  possible after being set.

* Supports migrations via the MGB (migration-begin), MGA (migration-abort) and MGE
  (migration-end) commands. The nodes automatically redistribute the keys taking
  the newly added ones into account.

  While the migration is in progress (and keys are being redistributed) the behaviour
  is the following:

  - If a get operation arrives, first the old continuum is checked, if not found
    the new continuum is checked.

  - If a set/delete operation arrives the new continuum is used to determine the
    owner of the key.

  Once the migration is completed the continua are swapped and the new continuum
  becomes the main one.

* Supports volatile keys, which have an expiration time and will be automatically
  removed when expired. Note that such keys are always kept in memory, regardless of
  the storage type, and are never passed to the storage backend. During a migration
  the volatile keys eventually not owned by a node will NOT be forwarded to the new
  owner but will be instead expired "prematurely" (since they won't be anyway
  available anymore when switching to the new continuum)

### shardcached

* Implements an `libshardcache` node with an HTTP frontend.

* Allows to define ACLs to control which IP addresses can access which keys.

* Exposes instrumentation counters and the cache index.

* Supports custom mime-type mappings to use when serving items over HTTP.

* Supports in-memory only or persistant storage.

* Provides an API to specify custom pluggable storage backends. Current support
  includes:
  - sqlite
  - mysql
  - redis

* Supports migrations that can be initiated by new nodes at startup.

Getting Started
===============

Shardcache development is done on GitHub. The latest release can be downloaded
from the [GitHub releases page](https://github.com/Shardcached/shardcached/releases),
or the latest development code can be cloned from the
[repository](https://github.com/Shardcached/shardcached).

The project is written in C, so it requires a working build environment. To compile,
invoke `make` in the toplevel directory. The makefile will fetch or update the
dependencies and build the daemon.

### Example

Here is a minimal example on how to start a cache of just one node, and on how to
use the example command line shardcache client to set and get a test key.

    $ export SHC_HOSTS="peer1:0.0.0.0:9090"
    $ ./shardcached -n $SHC_HOSTS -m peer1
    $ echo value | deps/libshardcache/utils/shardcachec set key
    $ deps/libshardcache/utils/shardcachec get key
    value
    $

### Usage

    Usage: ./shardcached [OPTION]...
    Version: 1.0 (libshardcache: 1.1)
    Possible options:
        -a <access_log_file>  the path where to store the access_log file (defaults to './shardcached_access.log')
        -c <config_file>      the config file to load
        -d <plugins_path>     the path where to look for storage plugins (defaults to './')
        -f                    run in foreground
        -F                    force caching
        -H                    disable the HTTP frontend
        -i <interval>         change the time interval (in seconds) used to report internal stats via syslog (defaults to '0')
        -l <ip_address:port>  ip_address:port where to listen for incoming http connections
        -L                    enable lazy expiration
        -M                    sets the arc_mode in libshardcache to 'loose' (defaults to : 'strict')
        -E <expire_time>      set the expiration time for cached items (defaults to: 0)
        -e <conn_expire_time> the expiration time for a connection in the pool to trigger a NOOP (defaults to 5000)
        -r <mux_timeout_low>  set the low timeout passed to iomux_run() calls (in microsecs, defaults to: 100000)
        -R <mux_timeout_high> set the high timeout pssed to iomux_run() calls (in microsecs, defaults to: 500000)
        -b                    HTTP url basepath (optional, defaults to '')
        -B                    HTTP url baseadminpath (optional, defaults to '')
        -n <nodes>            list of nodes participating in the shardcache in the form : 'label:address:port,label2:address2:port2'
        -N                    no storage subsystem, use only the internal libshardcache volatile storage
        -m me                 the label of this node, to identify it among the ones participating in the shardcache
        -P <pipelining_max>   the maximum amount of requests to handle ahead on the same connection while still serving a response (defaults to: 64)
        -S                    shared secret used for message signing (defaults to : '')
        -s                    cache size in bytes (defaults to : '536870912')
        -T <tcp_timeout>      tcp timeout (in milliseconds) used for connections opened by libshardcache (defaults to '5000')
        -t <type>             storage type (available are : 'mem' and 'fs' (defaults to 'mem')
        -o <options>          comma-separated list of storage options (defaults to '')
        -u <username>         assume the identity of <username> (only when run as root)
        -v                    increase the log level (can be passed multiple times)
        -V                    output the version number and exit
        -w <num_workers>      number of shardcache worker threads (defaults to '10')
        -W <num_http_workers> number of http worker threads (defaults to '10')
        -x <nodes>            new list of nodes to migrate the shardcache to. The format to use is the same as for the '-n' option

           Builtin storage types:
             * mem            memory based storage
                Options:
                  - initial_table_size=<size>    the initial number of slots in the internal hashtable
                  - max_table_size=<size>        the maximum number of slots that the internal hashtable can be grown up to

             * fs             filesystem based storage
                Options:
                  - storage_path=<path>          the path where to store the keys/values on the filesystem
                  - tmp_path=<path>              the path to a temporary directory to use while new data is being uploaded


