# Fixing vCenter Postgres Archiver Service - Dead Postgres Replication Slot


## Of course I run Backups :lying_face:

This time I had luck with the outcome of my recent homelab crash. If I weren't able to fix my broken vCenter Server, as described in my [previous article](https://rguske.github.io/post/database-resurrection-reviving-postgres-on-vmware-vcenter/), I would have had to reinstall my vSphere (+ vSAN, + Tanzu) environment basically from scratch again.

Is this actually true?

Actually NO! Because if I would have configured my vSphere environment correctly, [vCenter Server file-based backup](https://docs.vmware.com/en/VMware-vSphere/8.0/vsphere-vcenter-installation/GUID-3EAED005-B0A3-40CF-B40D-85AD247D7EA4.html) were configured properly and I wouldn't have had to worry about the consequences in the end.

But, I **haven't configured backup** and therefore, I wanted to do it the right way now.

## VMware Postgres Archiver Service stopped

I logged in into the vCenter Appliance Management Interface (VAMI) and started configuring my backup. When I kicked-off the first backup operation manually, an error was thrown on me immediateley. The error was telling me that something is wrong with the Backup Service. Consequently, I check the responsible `VMware Postgres Archiver` service.

{{< image src="/img/posts/202402_fixing_postgres_archiver_service/202402_fixing_postgres_archiver_service.png" caption="Figure I: VMware Postgres Archiver Service" src-s="/img/posts/202402_fixing_postgres_archiver_service/202402_fixing_postgres_archiver_service.png" >}}

Turned out, mine was in `stopped` state. Trying to simply start the service manually via `service-control --start vmware-postgres-archiver` didn't do the trick (too easy!).

Do we have chatty service logs? Yes, the Postgres Archiver Service has a dedicated `stderr` log which can be queried.

`tail -f /var/log/vmware/vpostgres/pg_archiver.log.stderr`

```shell
2024-01-29T10:07:07.339Z DEBUG  pg_archiver Updated startup LSN using segment file "00000001000000010000008A.gz"
2024-01-29T10:07:07.340Z DEBUG  pg_archiver Updated startup LSN using segment file "000000010000000300000098.gz"
2024-01-29T10:07:07.340Z DEBUG  pg_archiver Updated startup LSN using segment file "00000001000000030000009C.gz"
2024-01-29T10:07:07.340Z DEBUG  pg_archiver Updated startup LSN using segment file "0000000100000003000000A6.gz"
2024-01-29T10:07:07.340Z DEBUG  pg_archiver Updated startup LSN using segment file "0000000100000003000000AB.gz"
2024-01-29T10:07:07.343Z DEBUG  pg_archiver Updated startup LSN using segment file "0000000100000003000000AD.gz.partial"
2024-01-29T10:07:07.360Z DEBUG  pg_archiver starting log streaming at 3/AD000000 (timeline 1)
2024-01-29T10:07:07.361Z ERROR  pg_archiver unexpected termination of replication stream: ERROR:  requested WAL segment 0000000100000003000000AD has already been removed
```

> `ERROR  pg_archiver unexpected termination of replication stream` :thinking:

I continued with inspecting the `postgresql.log` and found this:

```shell
tail -f /var/log/vmware/vpostgres/postgresql.log

[...]

2024-01-29 10:07:06.725 UTC 65b778ca.25a22 0 archiver archiver [local] 154146 3DETAIL:  User "archiver" has no password assigned.
        Connection matched pg_hba.conf line 72: "local        all             all                             scram-sha-256"
2024-01-29 10:07:06.725 UTC 65b778ca.25a22 0 archiver archiver [local] 154146 4LOG:  could not send data to client: Broken pipe
2024-01-29 10:07:06.747 UTC 65b778ca.25a28 0 [unknown] [unknown] [local] 154152 1LOG:  connection received: host=[local]
2024-01-29 10:07:06.749 UTC 65b778ca.25a28 0 [unknown] archiver [local] 154152 2LOG:  connection authenticated: identity="vpostgres" method=peer (/storage/db/vpostgres/pg_hba.conf:9)
2024-01-29 10:07:06.749 UTC 65b778ca.25a28 0 [unknown] archiver [local] 154152 3LOG:  replication connection authorized: user=archiver application_name=pg_archiver
2024-01-29 10:07:06.752 UTC 65b778ca.25a28 0 [unknown] archiver [local] 154152 4ERROR:  replication slot "vpg_archiver" already exists

[...]
```

> `ERROR:  replication slot "vpg_archiver" already exists`

I searched for `replication_slot` in the official [PostgreSQL documentation](https://www.postgresql.org/docs/14/view-pg-replication-slots.html) and found how to query the `pg_replication_slots`.

```shell
/opt/vmware/vpostgres/current/bin/psql -U postgres -d VCDB -c "select * from pg_replication_slots;"

  slot_name   | plugin | slot_type | datoid | database | temporary | active | active_pid | xmin | catalog_xmin | restart_lsn | confirmed_flush_lsn | wal_status | safe_wal_size | two_phase
--------------+--------+-----------+--------+----------+-----------+--------+------------+------+--------------+-------------+---------------------+------------+---------------+-----------
 vpg_archiver |        | physical  |        |          | f         | f      |            |      |              | 3/AD000000  |                     | reserved   |               | f
(1 row)
```

Interesting to me was the `boolean` in column `active`. It was `false`.

In PostgreSQL, the "active" column represents whether a replication slot is currently actively being used or not. An "`f`" in this column indicates that the replication slot is not active, meaning it is not currently being utilized. Conversely, a value of "t" would indicate that the replication slot is active and is currently being used.

I continued gathering information and found VMware internally the final hints.

### Removing the PG_Replication_Slot

{{< admonition warning "Disclaimer" true >}}
You should raise a ticket at the VMware support first, before interfere with critical components of the vCenter Server. Therefore, the next steps are for non-production environments only.
{{< /admonition >}}

The solution here would be to remove the `vpg_archiver` replication slot using `pg_drop_replication_slot`.

From the vCenter Server `shell` execute:

```shell
/opt/vmware/vpostgres/current/bin/psql -U postgres -d VCDB -c "select pg_drop_replication_slot('vpg_archiver');"

 pg_drop_replication_slot
--------------------------

(1 row)
```

Query the `pg_replication_slots` once more to validate its removal.

```shell
/opt/vmware/vpostgres/current/bin/psql -U postgres -d VCDB -c "select * from pg_replication_slots;"

 slot_name | plugin | slot_type | datoid | database | temporary | active | active_pid | xmin | catalog_xmin | restart_lsn | confirmed_flush_lsn | wal_status | safe_wal_size | two_phase
-----------+--------+-----------+--------+----------+-----------+--------+------------+------+--------------+-------------+---------------------+------------+---------------+-----------
(0 rows)
```

As a final step before trying to restart the `vmware-postgres-archiver` service, is the deletion of already existing `segments` within `/storage/archive/vpostgres/`.

```shell
root@vcsa [ ~ ] df -h /storage/archive/vpostgres/

Filesystem                      Size  Used Avail Use% Mounted on
/dev/mapper/archive_vg-archive   49G  4.8G   42G  11% /storage/archive
```

Remove it: `rm /storage/archive/vpostgres/*`

Start the `vmware-postgres-archiver` service:

```shell
service-control --start vmware-postgres-archiver

Operation not cancellable. Please wait for it to finish...
Performing start operation on service vmware-postgres-archiver...
Successfully started service vmware-postgres-archiver
```

After bringing the service back into operating state, I started a new backup.

{{< image src="/img/posts/202402_fixing_postgres_archiver_service/202402_fixing_postgres_archiver_service_backup.png" caption="Figure II: VMware Postgres Archiver Service" src-s="/img/posts/202402_fixing_postgres_archiver_service/202402_fixing_postgres_archiver_service_backup.png" >}}

As you can see on *Figure II*, it took a few attempts but ultimately, it works again. Maybe the VCSA needed a moment to sort things out.

