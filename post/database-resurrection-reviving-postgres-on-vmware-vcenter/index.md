# Database Resurrection - Reviving vPostgres DB on VMware vCenter Server


## Power Failure causes Problems again

Once in a while power failures happen and can (mostly will!) cause troubles for small homelabs like mine. I'm running a two-node vSAN cluster on two Supermicro SYS-E200-8D servers. My vSAN Witness Appliance is running on a small Intel NUC, which is perfectly suited for this use case. The vCenter Server is still running on the vSAN cluster but only compute-wise. I have a Synology NAS running as well which is providing an additional NFS datastore in order to have at least the VCSA (VM) storage outsourced from the cluster.

However, if a power outage happen nothing will protect my lab from falling down hard. So, I definitely need a proper UPS battery soon if I decide to keep continuing running my lab. But! If I had already one, I couldn't write articles like this one, and [this](https://rguske.github.io/post/vmware-vctl-kind-writing-configuration-failed/) one and [this](https://rguske.github.io/post/fixing-vcenter-postgres-archiver-service-dead-replication-slot/) one :smile:

## VMware-vPostgres Service won't start anymore

After checking the cause of the outage and making sure it won't happen again, I brought everything back in running state. I logged into the vSphere Client and immediately faced the consequences. `The vSphere inventory objects can't be displayed`.

My first instinct was checking the state of all vCenter services via the vCenter Appliance Management Interface (VAMI). And here we go :fearful:

Most of the services kept staying in `stopped` state. I roughly knew the right start-order, or dependency-order of the affected services and checked them. One of the first services I tried starting was the `VMware Postgres` service. It almost immediately failed with a not really informative error. `Operation failed!...`

Consequently, I `ssh`ed into my VCSA and continued troubleshooting from there.

Same try, different approach. I tried starting the service using the `service-control` CLI.

```json
service-control --start vpostgres
Operation not cancellable. Please wait for it to finish...
Performing start operation on service vmware-vpostgres...
Error executing start on service vmware-vpostgres. Details {
    "resolution": null,
    "componentKey": null,
    "problemId": null,
    "detail": [
        {
            "translatable": "An error occurred while starting service '%(0)s'",
            "args": [
                "vmware-vpostgres"
            ],
            "id": "install.ciscommon.service.failstart",
            "localized": "An error occurred while starting service 'vmware-vpostgres'"
        }
    ]
}
Service-control failed. Error: {
    "resolution": null,
    "componentKey": null,
    "problemId": null,
    "detail": [
        {
            "translatable": "An error occurred while starting service '%(0)s'",
            "args": [
                "vmware-vpostgres"
            ],
            "id": "install.ciscommon.service.failstart",
            "localized": "An error occurred while starting service 'vmware-vpostgres'"
        }
    ]
```

Before I executed the `service-control` command, I started a second connection to my VCSA (I'm a fan of using the window-multiplexer [tmux](https://github.com/tmux/tmux/wiki)) and ran a `tail -f` on the vCenter service `vpxd` log. Here's a snipped from the log:

```code
[...]

2024-01-29T08:12:39.441Z error vpxd[04641] [Originator@6876 sub=vpxdVdb] [VpxdVdb::SetDBType] Failed to connect to database: ODBC error: (08001) - [unixODBC]connection to server on socket "/var/run/vpostgres/.s.PGSQL.5432" failed: No such file or directory
-->     Is the server running locally and accepting connections on that socket?
--> .  Retry attempt: 479617 ...
2024-01-29T08:12:39.451Z info vpxd[04641] [Originator@6876 sub=vpxdVdb] [VpxdVdb::SetDBType] Logging in to DSN: VMware VirtualCenter with username vpxd
2024-01-29T08:12:39.454Z info vpxd[04641] [Originator@6876 sub=vpxdVdb] [VpxdVdb::SetDBType] Logging in to DSN: VMware VirtualCenter with username vpxd
2024-01-29T08:12:39.456Z info vpxd[04641] [Originator@6876 sub=vpxdVdb] [VpxdVdb::SetDBType] Logging in to DSN: VMware VirtualCenter with username vpxd
2024-01-29T08:12:39.459Z info vpxd[04641] [Originator@6876 sub=vpxdVdb] [VpxdVdb::SetDBType] Logging in to DSN: VMware VirtualCenter with username vpxd
2024-01-29T08:12:39.463Z info vpxd[04641] [Originator@6876 sub=vpxdVdb] [VpxdVdb::SetDBType] Logging in to DSN: VMware VirtualCenter with username vpxd
2024-01-29T08:12:39.466Z info vpxd[04641] [Originator@6876 sub=vpxdVdb] [VpxdVdb::SetDBType] Logging in to DSN: VMware VirtualCenter with username vpxd
2024-01-29T08:12:39.469Z info vpxd[04641] [Originator@6876 sub=vpxdVdb] [VpxdVdb::SetDBType] Logging in to DSN: VMware VirtualCenter with username vpxd
2024-01-29T08:12:39.472Z info vpxd[04641] [Originator@6876 sub=vpxdVdb] [VpxdVdb::SetDBType] Logging in to DSN: VMware VirtualCenter with username vpxd
2024-01-29T08:12:39.476Z info vpxd[04641] [Originator@6876 sub=vpxdVdb] [VpxdVdb::SetDBType] Logging in to DSN: VMware VirtualCenter with username vpxd
2024-01-29T08:12:39.479Z info vpxd[04641] [Originator@6876 sub=vpxdVdb] [VpxdVdb::SetDBType] Logging in to DSN: VMware VirtualCenter with username vpxd
2024-01-29T08:12:39.481Z info vpxd[04641] [Originator@6876 sub=vpxdVdb] [VpxdVdb::SetDBType] Logging in to DSN: VMware VirtualCenter with username vpxd
2024-01-29T08:12:39.483Z info vpxd[04641] [Originator@6876 sub=vpxdVdb] [VpxdVdb::SetDBType] Logging in to DSN: VMware VirtualCenter with username vpxd
2024-01-29T08:12:39.486Z info vpxd[04641] [Originator@6876 sub=vpxdVdb] [VpxdVdb::SetDBType] Logging in to DSN: VMware VirtualCenter with username vpxd
2024-01-29T08:12:39.488Z info vpxd[04641] [Originator@6876 sub=vpxdVdb] [VpxdVdb::SetDBType] Logging in to DSN: VMware VirtualCenter with username vpxd
2024-01-29T08:12:39.491Z info vpxd[04641] [Originator@6876 sub=vpxdVdb] [VpxdVdb::SetDBType] Logging in to DSN: VMware VirtualCenter with username vpxd
^C
```

It don't need a PHD to get a clue that the error `Failed to connect to database: ODBC error: (08001) - [unixODBC]connection to server on socket "/var/run/vpostgres/.s.PGSQL.5432" failed: No such file or directory` isn't good at all.

Okay, my vPostgres instance got affected badly by the "unplanned shutdown".

## Reset to the last known working Checkpoint

{{< admonition warning "Disclaimer" true >}}
You should raise a ticket at the VMware support first, before intervene with critical components of the vCenter Server. Therefore, the next steps are more for lab environments.
{{< /admonition >}}

I started searching on the internet for appropriate tips and tricks but only found a couple of mainly outdated hints. Better than nothing, right?! My first attempt consisted of checking if I still can connect to the VCSA vPostgres instance using the postgres `psql` CLI.

```shell
root@vcsa [ ~ ] /opt/vmware/vpostgres/current/bin/psql -U postgres -d VCDB

psql.bin: error: connection to server on socket "/var/run/vpostgres/.s.PGSQL.5432" failed: No such file or directory
        Is the server running locally and accepting connections on that socket?
```

Well, it's not getting better. One would say, it's getting worse! I kept on searching but this time more Postgres related topics like "cannot connect postgres", "connection to server on socket..." or simply for "Postgres health check".

Luckily, I found out about the write-ahead log (WAL). I made myself familiar with it and decided to reset the `WAL` to the last working checkpoint.

{{< admonition info "Postgres WAL (Write-Ahead Log)" true >}}
The Postgres WAL (Write-Ahead Log) is the location in the Postgres cluster where all changes to the cluster's data files are recorded before they're written to the heap. When recovering from a crash, the WAL contains enough data for Postgres to restore its state to the last committed transaction.
{{< /admonition >}}

### 1st attempt pg_resetxlog

I started with:

```shell
root@vcsa [ ~ ] su vpostgres -s /opt/vmware/vpostgres/current/bin/pg_resetxlog /storage/db/vpostgres
Cannot execute /opt/vmware/vpostgres/current/bin/pg_resetxlog
```

Okay, this kind of message was unexpected but I quickly found out, that the command `pg_resetxlog` is for Postgres versions prior 10. It got [renamed](https://www.postgresql.org/docs/12/app-pgresetxlog.html) in v10 to `pg_resetwal`.

I'm already running vSphere 8 Update 2 and the used Postgres version here is 14.

### 2nd attempt pg_resetwal

Now, let's give the right command for the right version a try.

Start a `sh`ell session:

```shell
root@vcsa [ ~ ] su vpostgres -s /bin/sh
```

Change into the dir where the binary is located:

```shell
sh-5.0$ cd /opt/vmware/vpostgres/14/bin/
```

Unleash the process:

```shell
sh-5.0$ ./pg_resetwal /storage/db/vpostgres
The database server was not shut down cleanly.
Resetting the write-ahead log might cause data to be lost.
If you want to proceed anyway, use -f to force reset.
```

May the `force` be with you:

```shell
sh-5.0$ ./pg_resetwal /storage/db/vpostgres -f
Write-ahead log reset
sh-5.0$ exit
exit
```

Done! Check starting the `vmware-vpostgres` once again.

```shell
root@vcsa [ ~ ] service-control --start vmware-vpostgres
Operation not cancellable. Please wait for it to finish...
Performing start operation on service vmware-vpostgres...
Successfully started service vmware-vpostgres
```

`Successfully started service vmware-vpostgres` - hell yea!

Consequently, the rest of the services have to follow accordingly:

```shell
root@vcsa [ ~ ] service-control --start --all
Operation not cancellable. Please wait for it to finish...
Performing start operation on service lwsmd...
Successfully started service lwsmd
Performing start operation on service vmafdd...
Successfully started service vmafdd
Performing start operation on service vmdird...
Successfully started service vmdird
Performing start operation on service vmcad...
Successfully started service vmcad
Performing start operation on profile: ALL...
Successfully started profile: ALL.
Performing start operation on service observability...
Successfully started service observability
Performing start operation on service vmware-vdtc...
Successfully started service vmware-vdtc
Performing start operation on service vmware-pod...
Successfully started service vmware-pod
```

vSphere back up and running 🥳

