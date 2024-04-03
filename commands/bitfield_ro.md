Read-only variant of the `BITFIELD` command.
It is like the original `BITFIELD` but only accepts `!GET` subcommand and can safely be used in read-only replicas.

Since the original `BITFIELD` has `!SET` and `!INCRBY` options it is technically flagged as a writing command in the Valkey command table.
For this reason read-only replicas in a Valkey Cluster will redirect it to the master instance even if the connection is in read-only mode (see the `READONLY` command of Valkey Cluster).

See original `BITFIELD` for more details.

```
BITFIELD_RO hello GET i8 16
```
