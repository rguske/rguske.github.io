# Rebuilding from the Break - Restoring the VMware Avi Controller


## Avi Portal - Bad Service

Oh my! I wasn't expecting a three-part series as an outcome of my recent homelab crash, but here we are :smile: Check out my recent posts [Database Resurrection - Reviving vPostgres DB on VMware vCenter Server](https://rguske.github.io/post/database-resurrection-reviving-postgres-on-vmware-vcenter/) and [Fixing vCenter Postgres Archiver Service - Dead Postgres Replication Slot](https://rguske.github.io/post/fixing-vcenter-postgres-archiver-service-dead-replication-slot/) on my made experiences with recovering the vCenter Server database.

Actually, I thought I was "out of the woods" with fixiing the broken but unfortunately my Avi Controller was also affected from the outage.

When I tried accessing the portal, I got a `Bad service` error (see :point_up_2:).

First troubleshooting actions like e.g. using a different browser, clearing the browser-cache and cookies, etc. led to nothing.

Also "*Have you turn it on and off again* :nerd:" didn't help.

## Troubleshooting the System

Consequently, I established a connection via `ssh` to the Avi controller directly and checked its condition on the system-level.

```shell
ssh admin@avi-ctrl.jarvis.lab

Avi Cloud Controller

Avi Networks software, Copyright (C) 2013-2017 by Avi Networks, Inc.
All rights reserved.

Management:   10.10.70.10/24      UP
Gateway:      10.10.70.1          UP

admin@avi-ctrl.jarvis.lab's password:

# The copyrights to certain works contained in this software are
# owned by other third parties and used and distributed under
# license. Certain components of this software are licensed under
# the GNU General Public License (GPL) version 2.0 or the GNU
# Lesser General Public License (LGPL) Version 2.1. A copy of each
# such license is available at
# http://www.opensource.org/licenses/gpl-2.0.php and
# http://www.opensource.org/licenses/lgpl-2.1.php

Last login: Tue Feb  6 16:19:05 2024
```

I ran `journalctl` to check the systemd journal and my terminal filled up with errors like:

```shell
$ journalctl

[...]

-- Logs begin at Tue 2024-02-06 18:18:43 UTC, end at Wed 2024-02-07 07:06:22 UTC. --
Feb 06 18:18:43 10-10-70-10 sshd[902]: error: connect_to localhost port 5025: failed.

[...]
```

> `error: connect_to localhost port 5025: failed.`

The port number itself varied per entry. I looked it up on the internet but I couldn't find any useful information.

Next, I checked with the system and service manager `systemd` the conditions of the units (services, etc.) which are supposed to be up and running but it was a dead-end too. Only one unit was in state `failed` but I putted it aside, since it was only the `motd-news.service`:

```shell
systemctl list-units --state=failed

  UNIT              LOAD   ACTIVE SUB    DESCRIPTION
● motd-news.service loaded failed failed Message of the Day

LOAD   = Reflects whether the unit definition was properly loaded.
ACTIVE = The high-level unit activation state, i.e. generalization of SUB.
SUB    = The low-level unit activation state, values depend on unit type.

1 loaded units listed.
```

I continued digging around on the issue and read logs, checked on VMware's Knowledge Base (KB) and other internal resources but unfortunately in vain. I started realizing that I faced a low-level corruption of the system which can't be fixed in a simple fashion.

My suspicion got even more supported when I tried accessing the [Avi Shell](https://avinetworks.com/docs/22.1/cli-guide/):

```shell
admin@10-10-70-10:~$ shell --address 10.10.70.10
Login: admin
Password:

The controller at ['https://10.10.70.10'] is not active. If this is not the controller address, please restart the shell using the --address flag to specify the address of the controller.
{"error": "Bad service"}
```

---

{{< admonition info "Bonus - Docker run Avi Shell" true >}}
In section [Bonus: Docker run Avi Shell](#bonus-docker-run-avi-shell) I'm providing a brief guidance on how to run the Avi `shell` inside a container which you can easily start and use from basically any remote system which provides a container runtime.
{{< /admonition >}}

---

Problem is, that simply deploying a new instance a starting from scratch (w/o restoring data) isn't really an option, because other lab (business :wink:) critical systems relying on the Avi controller - on the Load Balancer service.

My Supervisor cluster (vSphere with Tanzu (TKGS)) for example:

{{< image src="/img/posts/202402_recover_avi_ctrl/202402_recover_avi_ctrl_sv_down.png" caption="Figure I: Supervisor Service affected" src-s="/img/posts/202402_recover_avi_ctrl/202402_recover_avi_ctrl_sv_down.png" >}}

## Backups FTW!

Well, fixing the Avi controller manually on the system-level wasn't an option anymore, therefore **restoring** the "Good" from a backup was next on the list.

By default, the Avi controller runs backups periodically and stores the files locally in `/var/lib/avi/backups`. You might remember that you've configured the passphrase for the backups as one of the first steps when you've deployed the controller.

In a working instance, the configuration in terms of frequency, retention and remote backup location can be adjusted under **Administration - Configuration Backup**.

{{< image src="/img/posts/202402_recover_avi_ctrl/202402_recover_avi_ctrl_backup_conf.png" caption="Figure II: Avi Controller Backup Configuration" src-s="/img/posts/202402_recover_avi_ctrl/202402_recover_avi_ctrl_backup_conf.png" >}}

Per default, the backup is configured to run every 10 minutes and has a retention of 4 files. Best practice is of course to additionally store (copy) the backup-files on a remote server location using either `SCP` or `SFTP`.

I recommend looking up the official documentation for more options on backing up the Avi configuraiton. The documentation contains great examples on how to configure the Avi backup using e.g. the `avi_shell`, the Avi API, REST API or even via Python.

:book: [Backup and Restore of Avi Vantage Configuration](https://docs.vmware.com/en/VMware-NSX-Advanced-Load-Balancer/22.1/Administration_Guide/GUID-0CE79F00-ECC3-49AE-B5F5-004D1A835400.html). Alternatively, check `origin` [HERE](https://avinetworks.com/docs/22.1/backup-and-restore-of-avi-vantage-configuration/).

## Restoring the Avi Controller

I had a plan!

### Backup the Backup

My plan included the following:

1. Copy the files to my computer to have a copy of an existing backup.
2. Run the `restore_config.py` script with the `--flushdb` option as descriped in the docs - [Restore the VMware NSX Advanced Load Balancer Configuration](https://docs.vmware.com/en/VMware-NSX-Advanced-Load-Balancer/22.1/Administration_Guide/GUID-B300E74E-F366-46D7-99C5-AB64961FCC48.html).
3. Celebrating the restored instance.

**Let's go!**

Like I've mentioned earlier, the backup-files are stored locally in `/var/lib/avi/backups`:

```shell
ls -rtl
total 10904
-rw-r--r-- 1 root root 3040701 Jan 26 11:00 backup_Default-Scheduler_20240126_110041.json
-rw-r--r-- 1 root root 3044239 Feb  1 11:00 backup_Default-Scheduler_20240201_110048.json
-rw-r--r-- 1 root root 2536819 Feb  2 11:00 backup_Default-Scheduler_20240202_110041.json
-rw-r--r-- 1 root root 2533528 Feb  3 11:00 backup_Default-Scheduler_20240203_110044.json
```

Since I was still able to connect to my Avi controller via `ssh`, I was also able to remote-copy (`scp`) the files.

I started packaging the files into a tar-ball:

```shell
sudo tar -czf /tmp/backup.tar *.json

ls -rtl /tmp/backup.tar
-rw-r--r-- 1 root root 18042880 Feb 29 10:28 /tmp/backup.tar
```

Next was "downloading" the files to my computer:

```shell
scp admin@10.10.70.10:/tmp/backup.tar ~/Downloads/avibackup

Avi Cloud Controller

Avi Networks software, Copyright (C) 2013-2017 by Avi Networks, Inc.
All rights reserved.

Management:   10.10.70.10/24      UP
Gateway:      10.10.70.1          UP

admin@10.10.70.10's password:
backup.tar                                 100%   17MB  80.6MB/s   00:00
```

{{< admonition info "BONUS - Powershell and Bash Scripts available" true >}}
I've created a Powershell as well a Bash script to simplify the download of the backup-files from the Avi Controller. Both scripts can be copied from [here](#bonus-powershell-and-bash-scripts-to-download-avi-backup-files).
{{< /admonition >}}

### The Restore Process

Now that I have a backup of my backup, I was brave enough to execute the `restore_config.py` script with the option `--flushdb`.

BTW! If you don't use the option, the restore script will immediately end with the error:

> `Restore config must be run on a clean controller. Please run with the --flushdb option or do a reboot clean to clear all config before restoring.`

Executing `restore_config.py`:

```shell
sudo /opt/avi/scripts/restore_config.py --config /var/lib/avi/backups/backup_Default-Scheduler_20240203_110044.json --passphrase 'VMware1!' --flushdb

Traceback (most recent call last):
  File "/opt/avi/scripts/restore_config.py", line 98, in <module>
    user, token = get_username_token()
  File "/opt/avi/python/lib/avi/util/login.py", line 136, in get_username_token
    rsp = json.loads(response.stdout)
  File "/usr/lib/python3.8/json/__init__.py", line 357, in loads
    return _default_decoder.decode(s)
  File "/usr/lib/python3.8/json/decoder.py", line 340, in decode
    raise JSONDecodeError("Extra data", s, end)
json.decoder.JSONDecodeError: Extra data: line 1 column 9 (char 8)

restore_config.py on node (10-10-70-10) crash -- Pls investigate further.
core_archive.20240207_152903.tar.gz not collected due to similar stack-trace se
en=restore_config.py.20240207_152728.stack_trace
```

:thinking: Not what I expected!

I tried it with other files once again but without success. I also tried executing the `flush_db.py` script alone:

```shell
admin@10-10-70-10:/opt/avi/scripts$ sudo ./flushdb.sh
rm: cannot remove '/etc/luna/cert/server/*': No such file or directory
rm: cannot remove '/etc/luna/cert/client/*': No such file or directory
debconf: unable to initialize frontend: Dialog
debconf: (No usable dialog-like program is installed, so the dialog based frontend cannot be used. at /usr/share/perl5/Debconf/FrontEnd/Dialog.pm line 76.)
debconf: falling back to frontend: Readline
Creating SSH2 RSA key; this may take some time ...
3072 SHA256:OZDiQ2SqCXaWR5+Dd1PLc95ekDY6Ix5el0JPAHoc9Ck root@10-10-70-10 (RSA)
rescue-ssh.target is a disabled or a static unit, not starting it.
```

But it also failed.

Well, if the `flush_db` option erases the controller config anyway, then deploying a fresh instance is an option too.

### Deploying a fresh Instance

So, I deployed a new instance of the Avi controller (OVA), `scp`ed the existing backup-files to the new controller, connected via `ssh` and executed the `restore_config.py` once again.

```shell
sudo /opt/avi/scripts/restore_config.py --config /tmp/backup_Default-Scheduler_20240203_110044.json --passphrase VMware1! --flushdb

Restoring config will duplicate the environment of the config. If the environment is active elsewhere there will be conflicts.
                Please confirm if you still want to continue (yes/no)yes

======== Configuration Restored for 1-node cluster ========
```

**Note:** The documentation is mentioning options like:

- `--do_not_form_cluster`
- `--vip`
- `--followers`

These options are not supported in version 22.1.3 and onwards anymore.

If you are running the Avi controller in cluster-mode (three-nodes), roll out the other two nodes as well, configure the same IP-addresses as you have used before and simply add them to the Controller Cluster using the gui **Administration -> Controller -> Nodes -> Edit**.

It is not intended to execute the restore process on the `follower` nodes!. They'll get the configuration mirrored from the `leader`.

## Back on the Job

Restoring the Avi controller is really a simple and effective procedure. Ultimately, troubleshooting my broken instance costed me more time than simply restoring a new one.

But we won't be engineers if we wouldn't try to fix the broken, right? On the bright sight, additionally to this article, two scripts and a way how to run the `avi_shell` in a container on your computer are the outcome.

Here we go!

{{< image src="/img/posts/202402_recover_avi_ctrl/202402_recover_avi_ctrl_sv_healthy.png" caption="Figure III: vSphere with Tanzu Supervisor Service is back in healthy condition" src-s="/img/posts/202402_recover_avi_ctrl/202402_recover_avi_ctrl_sv_healthy.png" >}}

## Bonus: Docker run Avi Shell

In the [first section](#troubleshooting-the-system) of my article, I mentioned that I wasn't able to open the Avi Shell from within the controller. When I entered the command `shell` in order to start it, I received the error:

```shell
admin@10-10-70-10:~$ shell
Login: admin
Password:

The controller at ['https://10.10.70.10'] is not active. If this is not the controller address, please restart the shell using the --address flag to specify the address of the controller.
{"error": "Bad service"}
```

I also tried it with the option `--address` but unfortunately without changing the result. Interesting is, that when entering `shell` on the controller, a container will spin up in the background and will provide the `shell`.

This got me thinking. It can come in handy if you can quickly make a shell available in a container without having to install the `shell` itself on a system.

Therefore, here's a way to run a container on your system which will host the Avi `shell`:

- Identify the container image on the Avi controller using `docker images`:

```shell
admin@10-10-70-10:~$ sudo docker images

[sudo] password for admin:
REPOSITORY                  TAG           IMAGE ID       CREATED        SIZE
avinetworks/cli             22.1.5.9093   8fd02452dac3   4 months ago   484MB
avinetworks/controlscript   22.1.5.9093   ecc7d8244a83   7 months ago   72.8MB
avinetworks/grafana         influxdb3     4a4cc4a8f5e6   3 years ago    159MB
avinetworks/grafana         latest        38e8ff91a07e   4 years ago    246MB
```

- Package the container image into a tar-ball using `docker save`:

```shell
admin@10-10-70-10:~$ sudo docker save avinetworks/cli -o /tmp/avinetworks-cli.tar

admin@10-10-70-10:~$ ls -l /tmp/avinetworks-cli.tar
-rw------- 1 root root 500762624 Feb 28 14:50 /tmp/avinetworks-cli.tar
```

- Give the Avi default user `admin` owner permissions to the tar-ball:

```shell
admin@10-10-70-10:~$ sudo chown admin /tmp/avinetworks-cli.tar

admin@10-10-70-10:~/avicli-image$ ls -l
total 489032
-rw------- 1 admin root 500762624 Feb 28 14:45 avinetworks-cli.tar
```

- Download the tar-ball from the Avi controller:

```shell
scp admin@10.10.70.10:/tmp/avinetworks-cli.tar ~/Downloads/

Avi Cloud Controller

Avi Networks software, Copyright (C) 2013-2017 by Avi Networks, Inc.
All rights reserved.

Management:   10.10.70.10/24      UP
Gateway:      10.10.70.1          UP

admin@10.10.70.10's password:
avinetworks-cli.                    100%  478MB  35.8MB/s   00:13
```

- Import the container image to your system using `docker load`:

```shell
docker load -i avinetworks-cli.tar
```

- Check its availability:

```shell
docker images | grep avinetworks
avinetworks/cli    22.1.5.9093         8fd02452dac3   4 months ago    484MB
```

- Run the Avi `shell` in a container and directly connect to your Avi controller:

```shell
docker run -it --rm --name avishell  avinetworks/cli:22.1.5.9093 /usr/local/bin/avi_shell --address 10.10.70.10 --user admin --password 'VMware1!'

[admin:10-10-70-10]: > show backup
+---------------------------------------------+
| UUID                                        |
+---------------------------------------------+
| backup-e97d3c3e-ade9-4249-8a94-4e77d358d74d |
| backup-36bdd91c-b952-4644-82eb-1138803a1e04 |
| backup-329c1194-8073-4acb-82d0-41b976e60668 |
| backup-e14992c7-b2e1-4ff0-9fb3-ca760bc60203 |
+---------------------------------------------+

[admin:10-10-70-10]: > show backupconfiguration Backup-Configuration
+-------------------------------+----------------------------------------------------------+
| Field                         | Value                                                    |
+-------------------------------+----------------------------------------------------------+
| uuid                          | backupconfiguration-34c64416-83d8-4f56-9775-f66489fe80ce |
| name                          | Backup-Configuration                                     |
| save_local                    | True                                                     |
| maximum_backups_stored        | 4                                                        |
| backup_passphrase             | <sensitive>                                              |
| remote_file_transfer_protocol | SCP                                                      |
| tenant_ref                    | admin                                                    |
+-------------------------------+----------------------------------------------------------+
[admin:10-10-70-10]: >
```

The Avi `shell` can be very usefull when you'd like to obtain specific information or if you'd like to make adjustments on the existing configuration to the Avi controller without using the GUI.

Check out the [CLI Top-Level Commands](https://avinetworks.com/docs/latest/cli-top-level-commands/).

**Examples**:

```shell
[admin:10-10-70-10]: > show virtualservice
+--------------------------------------------------------+-------------+-------------+-----------+---------------+------------+----------------+--------+
| Name                                                   | IP Address  | IP6 Address | Services  | Cloud         | Oper State | Type           | Parent |
+--------------------------------------------------------+-------------+-------------+-----------+---------------+------------+----------------+--------+
| domain-c1015--ns-mk1-rguske-mk1-control-plane-service  | 10.10.70.56 | -           | 6443      | Default-Cloud | OPER_UP    | VS_TYPE_NORMAL | -      |
| domain-c1015--svc-contour-domain-c1015-envoy           | 10.10.70.54 | -           | 80,443    | Default-Cloud | OPER_UP    | VS_TYPE_NORMAL | -      |
| domain-c1015--kube-system-kube-apiserver-lb-svc        | 10.10.70.51 | -           | 443,6443  | Default-Cloud | OPER_UP    | VS_TYPE_NORMAL | -      |
| domain-c1015--vmware-system-csi-vsphere-csi-controller | 10.10.70.50 | -           | 2112,2113 | Default-Cloud | OPER_UP    | VS_TYPE_NORMAL | -      |
| domain-c1015--ns-mk2-rguske-mk2-control-plane-service  | 10.10.70.55 | -           | 6443      | Default-Cloud | OPER_UP    | VS_TYPE_NORMAL | -      |
| domain-c1015--ns-mk2-rguske-mk2-431dce4740524bc9173aa  | 10.10.70.57 | -           | 9000,9001 | Default-Cloud | OPER_UP    | VS_TYPE_NORMAL | -      |
+--------------------------------------------------------+-------------+-------------+-----------+---------------+------------+----------------+--------+
```

```shell
[admin:10-10-70-10]: > show placement vip detail

[...]

+---------------------------+-------------------------------------------------------+
| Field                     | Value                                                 |
+---------------------------+-------------------------------------------------------+
| vip                       | 10.10.70.55                                           |
| primary_se_info[1]        |                                                       |
|   se_uuid                 | se-02e46378-8271-478a-9264-c3ed893562a6               |
|   se_name                 | avi-se-ljvci                                          |
|   num_consumers_using_vip | 1                                                     |
| secondary_se_info[1]      |                                                       |
|   se_uuid                 | se-5b8f5b8b-39b3-45f0-8ec9-39c3a901eb2f               |
|   se_name                 | avi-se-tumzb                                          |
|   num_consumers_using_vip | 1                                                     |
| con_info[1]               |                                                       |
|   con_uuid                | virtualservice-9dfc3721-0fb2-4e93-918b-4df3f40ea991#0 |
|   con_name                | domain-c1015--ns-mk2-rguske-mk2-control-plane-service |
| vip6                      |                                                       |
| uuid                      | vsvip-badee2b7-c908-40bb-998b-295d23be77c9#0          |
+---------------------------+-------------------------------------------------------+
+---------------------------+-------------------------------------------------------+
| Field                     | Value                                                 |
+---------------------------+-------------------------------------------------------+
| vip                       | 10.10.70.56                                           |
| primary_se_info[1]        |                                                       |
|   se_uuid                 | se-5b8f5b8b-39b3-45f0-8ec9-39c3a901eb2f               |
|   se_name                 | avi-se-tumzb                                          |
|   num_consumers_using_vip | 1                                                     |
| secondary_se_info[1]      |                                                       |
|   se_uuid                 | se-02e46378-8271-478a-9264-c3ed893562a6               |
|   se_name                 | avi-se-ljvci                                          |
|   num_consumers_using_vip | 1                                                     |
| con_info[1]               |                                                       |
|   con_uuid                | virtualservice-2f08dfac-bb3d-4e5f-b928-5b23f5f97db0#0 |
|   con_name                | domain-c1015--ns-mk1-rguske-mk1-control-plane-service |
| vip6                      |                                                       |
| uuid                      | vsvip-e5e99c90-1402-4425-b495-6e4ef56ae9cd#0          |
+---------------------------+-------------------------------------------------------+
+---------------------------+-------------------------------------------------------+
| Field                     | Value                                                 |
+---------------------------+-------------------------------------------------------+
| vip                       | 10.10.70.57                                           |
| primary_se_info[1]        |                                                       |
|   se_uuid                 | se-02e46378-8271-478a-9264-c3ed893562a6               |
|   se_name                 | avi-se-ljvci                                          |
|   num_consumers_using_vip | 1                                                     |
| secondary_se_info[1]      |                                                       |
|   se_uuid                 | se-5b8f5b8b-39b3-45f0-8ec9-39c3a901eb2f               |
|   se_name                 | avi-se-tumzb                                          |
|   num_consumers_using_vip | 1                                                     |
| con_info[1]               |                                                       |
|   con_uuid                | virtualservice-dae117d8-eaad-4ebf-86b7-d99c6f2ecc53#0 |
|   con_name                | domain-c1015--ns-mk2-rguske-mk2-431dce4740524bc9173aa |
| vip6                      |                                                       |
| uuid                      | vsvip-e86cdd4e-bde1-4222-9384-eb85dba5d8ec#0          |
+---------------------------+-------------------------------------------------------+
```

## Bonus: Powershell and Bash Scripts to download Avi Backup Files

Powershell (with `pscp.exe`):

```powershell
# Define the variables
$remoteServer = "admin@10.10.70.10"
$remotePath = "/var/lib/avi/backups/*"
$baseLocalPath = "C:\avibackups"
# Download pscp from: https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html
$pscpPath = "C:\avibackups\pscp.exe"

# Generate a timestamp
$timestamp = Get-Date -Format "yyyy-MM-dd_HH-mm-ss"

# Create a new directory with the timestamp
$newLocalPath = Join-Path -Path ${baseLocalPath} -ChildPath ${timestamp}
New-Item -ItemType Directory -Force -Path ${newLocalPath}

# Optional: Add your SSH key path if you're using one
# $sshKeyPath = "C:\path\to\your\private_key.ppk"

# Download files from remote server into the newly created directory using $sshKeyPath
# & $pscpPath -i ${sshKeyPath} ${remoteServer}:${remotePath} ${newLocalPath}

# Download files from remote server into the newly created directory
& $pscpPath ${remoteServer}:${remotePath} ${newLocalPath}
```

{{< image src="/img/posts/202402_recover_avi_ctrl/202402_recover_avi_ctrl_ps_backup.png" caption="Figure IV: Powershell Avi Config-files Backup" src-s="/img/posts/202402_recover_avi_ctrl/202402_recover_avi_ctrl_ps_backup.png" >}}

Bash:

```bash
#!/bin/bash
set -euo pipefail

# Define variables
REMOTE_SERVER="admin@10.10.70.10"
REMOTE_PATH="/var/lib/avi/backups/*"
BASE_LOCAL_PATH="/Users/rguske/Downloads/avibackup"

# Generate a timestamp
TIMESTAMP=$(date +"%Y-%m-%d_%H-%M-%S")

# Create a new directory with the timestamp
NEW_LOCAL_PATH="$BASE_LOCAL_PATH/$TIMESTAMP"
mkdir -p "$NEW_LOCAL_PATH"

# Download files from the remote server into the newly created directory
scp -r "${REMOTE_SERVER}:${REMOTE_PATH}" "${NEW_LOCAL_PATH}"
```

{{< image src="/img/posts/202402_recover_avi_ctrl/202402_recover_avi_ctrl_bash_backup.png" caption="Figure V: Bash Avi Config-files Backup" src-s="/img/posts/202402_recover_avi_ctrl/202402_recover_avi_ctrl_bash_backup.png" >}}

**Thanks for reading my article**

