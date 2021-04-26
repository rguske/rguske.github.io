# VMware ovftool installation was unsuccessful on Ubuntu 20.04 - A workaround


<!--more-->

## Introduction

The VMware OVF Tool [^1] is a powerful cli utility with which you can import and export Open Virtualization Format (OVF) packages to and from various VMware products. It's e.g. used for the creation of several awesome VMware Fling projects like the Demo Appliance for Tanzu Kubernetes Grid [^2], the VMware Appliance for Folding@Home [^3] as well as for the VMware Event Broker Appliance [^4]. Also non-fling open source projects like e.g. the Netshoot Virtual Appliance [^5] using the `ovftool` in their appliance build process. William Lam blogged about it several times in the past and you could use this as a great reference for the tool [^6].

I recently wanted to make use of the `ovftool` for the validation of a customer use-case and needed to install it on my "*Universal Workbench*" [^7], my Ubuntu 20.04 vDesk (`20.04.1-Ubuntu x86_64`), which I am using for all my homelab work. As of writing this article, the latest available version for the `ovftool` is version 4.4.1 [^8].

## Ovftool installation options

I downloaded the appropriate 64 bit version for my Linux system, which is the `VMware-ovftool-4.4.1-16812187-lin.x86_64.bundle` file and made it executable for the installation:

```shell
$ chmod +755 VMware-ovftool-4.4.1-16812187-lin.x86_64.bundle
```

The `ovftool` installation file is a bundled package (`.bundle`) which contains the `ovftool` as well as the `vmware-installer`, plus the associated files for those. The installation can be executed with various options and by running e.g. `./VMware-ovftool-4.4.1-16812187-lin.x86_64.bundle --help`, the `vmware-installer` will be temporarily extracted and provides these options. The following listed are an abstract of the ones which are relevant for this post and which I've used for my installation attempts.

```shell
Options:
    --gtk               Use the Gtk+ UI (Default)
    --console           Use the console UI
    --required          Displays only questions absolutely required
    --eulas-agreed      Agree to the EULA
    --extract=DIR       Extract the contents of the bundle into DIR
```

## Installation was unsuccessful

On Linux, you can choose either to use the `--gtk` UI (default) or the `--console` UI option. My first attempt, and I used this **several** times before, was with the `--console` option. In a Terminal window I executed `sudo ./VMware-ovftool-4.4.1-16812187-lin.x86_64.bundle --console --required --eulas-agreed` and got the following output as a failed result:

```shell
Extracting VMware Installer...done.
Traceback (most recent call last):
  File "/tmp/tmpt0pd6m09.vmis.env", line 132, in <module>
    exec(fileObj.read(), mod.__dict__)
  File "<string>", line 243
    except OSError, e:
                  ^
SyntaxError: invalid syntax
Exception info : [Component did not register an installer].
Check the log /var/log/vmware-installer for details.



[######################################################################] 100%
Installation was unsuccessful.
```

- <mark>*Installation was unsuccessful*</mark>

I couldn't really figure out the output and went straight for the `--gtk` option as my 2nd attempt:

```shell
$ sudo ./VMware-ovftool-4.4.1-16812187-lin.x86_64.bundle --gtk --required --eulas-agreed

Extracting VMware Installer...done.

(vmware-installer.py:233744): Gtk-WARNING **: Unable to locate theme engine in module_path: "adwaita",

(vmware-installer.py:233744): Gtk-WARNING **: Unable to locate theme engine in module_path: "adwaita",
Gtk-Message: Failed to load module "canberra-gtk-module"
Traceback (most recent call last):
  File "/tmp/tmp1nqoz5ld.vmis.env", line 132, in <module>
    exec(fileObj.read(), mod.__dict__)
  File "<string>", line 243
    except OSError, e:
                  ^
SyntaxError: invalid syntax
Exception info : [Component did not register an installer].
Check the log /var/log/vmware-installer for details.



[######################################################################] 100%
Installation was unsuccessful.
```

- <mark>*Gtk-WARNING: Unable to locate theme engine in module_path: adwaita*</mark>

- <mark>*Gtk-Message: Failed to load module canberra-gtk-module*</mark>

Some research on both failure indications pointed me to presumable missing packages related to the `GTK graphical user interface library` in Ubuntu and therefore I followed some of the described hints which were for example:

- `sudo apt install gnome-themes-standard`[^9]
- `sudo apt install canberra-gtk-module`[^10]

but non of those solved the problem nor changed the failure message. Also the `/var/log/vmware-installer` log couldn't really help (at least to me).

### Output /var/log/vmware-installer

```shell
$ less /var/log/vmware-installer


[2021-02-25 11:28:13,465] code for hash sha256 was not found.
Traceback (most recent call last):
  File "/tmp/vmis.Lu1nSs/install/vmware-installer/python/lib/hashlib.py", line 147, in <module>
    globals()[__func_name] = __get_hash(__func_name)
  File "/tmp/vmis.Lu1nSs/install/vmware-installer/python/lib/hashlib.py", line 97, in __get_builtin_constructor
    raise ValueError('unsupported hash type ' + name)
ValueError: unsupported hash type sha256
[2021-02-25 11:28:13,465] code for hash sha384 was not found.
Traceback (most recent call last):
  File "/tmp/vmis.Lu1nSs/install/vmware-installer/python/lib/hashlib.py", line 147, in <module>
    globals()[__func_name] = __get_hash(__func_name)
  File "/tmp/vmis.Lu1nSs/install/vmware-installer/python/lib/hashlib.py", line 97, in __get_builtin_constructor
    raise ValueError('unsupported hash type ' + name)
ValueError: unsupported hash type sha384
[2021-02-25 11:28:13,465] code for hash sha512 was not found.
Traceback (most recent call last):
  File "/tmp/vmis.Lu1nSs/install/vmware-installer/python/lib/hashlib.py", line 147, in <module>
    globals()[__func_name] = __get_hash(__func_name)
  File "/tmp/vmis.Lu1nSs/install/vmware-installer/python/lib/hashlib.py", line 97, in __get_builtin_constructor
    raise ValueError('unsupported hash type ' + name)
ValueError: unsupported hash type sha512
[2021-02-25 11:28:13,494]
[2021-02-25 11:28:13,494]
[2021-02-25 11:28:13,494] Installer running.
[2021-02-25 11:28:13,494] Command Line Arguments:
[2021-02-25 11:28:13,494] ['/tmp/vmis.Lu1nSs/install/vmware-installer/vmware-installer.py', '--set-setting', 'vmware-installer', 'libconf', '', '--install-component', '/tmp/vmis.Lu1nSs/install/vmware-installer', '--install-bundle', '/home/rguske/Downloads/./VMware-ovftool-4.4.0-16360108-lin.x86_64.bundle', '']
[2021-02-25 11:28:13,740] Successfully loaded GTK libraries.
[2021-02-25 11:28:13,789] Using UI type gtk
[2021-02-25 11:28:13,795] System installer version is: 3.0.0.17287072
[2021-02-25 11:28:13,795] Running installer version is: 2.1.0.14928104
[2021-02-25 11:28:13,795] Running newer system installer.
[2021-02-25 11:28:14,151]
[2021-02-25 11:28:14,151]
[2021-02-25 11:28:14,151] Installer running.
[2021-02-25 11:28:14,151] Command Line Arguments:
[2021-02-25 11:28:14,151] ['/usr/lib/vmware-installer/3.0.0/vmware-installer.py', '--set-setting', 'vmware-installer', 'libconf', '', '--install-component', '/tmp/vmis.Lu1nSs/install/vmware-installer', '--install-bundle', '/home/rguske/Downloads/./VMware-ovftool-4.4.0-16360108-lin.x86_64.bundle', '']
[2021-02-25 11:28:14,151] Failed to load GTK libraries, falling back to console.
[2021-02-25 11:28:14,157] Using UI type console
[2021-02-25 11:28:14,158] System installer version is: 3.0.0.17287072
[2021-02-25 11:28:14,158] Running installer version is: 3.0.0.17287072
[2021-02-25 11:28:14,158] Opening database file /etc/vmware-installer/database
[2021-02-25 11:28:14,236] Could not locate installer App Control.
[2021-02-25 11:28:14,247] Error: Cannot load installer for component: vmware-installer.
[2021-02-25 11:28:14,248] Top level exception handler
Traceback (most recent call last):
  File "/usr/lib/vmware-installer/3.0.0/vmis/core/transaction.py", line 470, in RunThreadedTransaction
    txn.Execute(actions)
  File "/usr/lib/vmware-installer/3.0.0/vmis/core/transaction.py", line 252, in Execute
    i.Load(self.temp)
  File "/usr/lib/vmware-installer/3.0.0/vmis/core/install.py", line 325, in Load
    self._module, self._installer = LoadInstaller(self.component,
  File "/usr/lib/vmware-installer/3.0.0/vmis/core/env.py", line 406, in LoadInstaller
    raise ComponentError('Component did not register an installer', component)
vmis.core.errors.ComponentError: Component did not register an installer
```

## The workaround

If we cannot reach our goal via the conventional way, we have to find a detour, a workaround to get there. For this particular issue with the failed installation on Ubuntu 20.04, it is the extraction of the `ovftool` files from the installation `.bundle`, copying them to `/usr/bin/` and configure an `alias` for the `ovftool` executable.

### Execute `ovftool` without installing it

**This is how it works step-by-step:**

1. Extract the files and change into directory ovftool (you can name the extracted directory as you want):

`sudo ./VMware-ovftool-4.4.1-16812187-lin.x86_64.bundle --extract ovftool && cd ovftool`

```shell
ls -rtl

total 16K
drwxr-xr-x 4 root   root         4,0K Mär  9 14:07 .
drwxr-xr-x 5 rguske domain^users 4,0K Mär  9 14:07 ..
drwxr-xr-x 9 root   root         4,0K Mär  9 14:07 vmware-installer
drwxr-xr-x 6 root   root         4,0K Mär  9 14:07 vmware-ovftool
```

Here's what's in it:

```shell
$ tree -L 2
.
├── vmware-installer
│   ├── artwork
│   ├── bin
│   ├── bootstrap
│   ├── lib
│   ├── manifest.xml
│   ├── python
│   ├── sopython
│   ├── vmis
│   ├── vmis-launcher
│   ├── vmware-installer
│   ├── vmware-installer.py
│   ├── vmware-uninstall
│   └── vmware-uninstall-downgrade
└── vmware-ovftool
    ├── certs
    ├── env
    ├── icudt44l.dat
    ├── libcares.so.2
    ├── libcrypto.so.1.0.2
    ├── libcurl.so.4
    ├── libexpat.so
    ├── libgcc_s.so.1
    ├── libgoogleurl.so.59
    ├── libicudata.so.60
    ├── libicuuc.so.60
    ├── libssl.so.1.0.2
    ├── libssoclient.so
    ├── libstdc++.so.6
    ├── libvim-types.so
    ├── libvmacore.so
    ├── libvmomi.so
    ├── libxerces-c-3.2.so
    ├── libz.so.1
    ├── manifest.xml
    ├── open_source_licenses.txt
    ├── ovftool
    ├── ovftool.bin
    ├── README.txt
    ├── schemas
    ├── vmware.eula
    └── vmware-eula.rtf

11 directories, 31 files
```

2. Move the `vmware-ovftool` directory to `/usr/bin/`:

```shell
sudo mv vmware-ovftool /usr/bin/
```

3. Make the two files `ovftool` as well as `ovftool.bin` executable:


```shell
sudo chmod +x /usr/bin/vmware-ovftool/ovftool ovftool.bin
```

4. Configure an `alias` for your used `shell` to execute it as usual: `alias ovftool=/usr/bin/vmware-ovftool/ovftool`

I am using `zsh` [^11] as the `shell`of my choice and I created a dedicated file for all my aliases - it's named `~/.zsh_aliases`.

```
$ which ovftool

ovftool: aliased to /usr/bin/vmware-ovftool/ovftool
```

This allows me to run the `ovftool` as usual.

## Conclusion

- Ovftool installation was unsuccessful and issued the following errors: `Unable to locate theme engine in module_path: "adwaita"` and `Failed to load module "canberra-gtk-module`
- Installation of Linux packages `gnome-themes-standard` and `canberra-gtk-module` didn't solve anything
- Extracted the `ovftool` binaries from the installation `.bundle` package by using `vmware-installer` option `--extract`
- Moved extracted folder `vmware-ovftool` to `/usr/bin/`
- Made files `ovftool` and `ovftool.bin` executable
- configured an `alias` for our shell to use it as usual


[^1]: [VMware ovftool {code} page](https://code.vmware.com/web/tool/4.4.0/ovf)
[^2]: [Demo Appliance for Tanzu Kubernetes Grid](https://flings.vmware.com/demo-appliance-for-tanzu-kubernetes-grid)
[^3]: [VMware Appliance for Folding@Home](https://flings.vmware.com/vmware-appliance-for-folding-home)
[^4]: [VMware Event Broker Appliance](https://vmweventbroker.io/kb/contribute-appliance)
[^5]: [Netshoot Virtual Appliance](https://github.com/josemzr/netshoot-virtual-appliance)
[^6]: [virtuallyGhetto - OVF / OVFTOOL](https://www.virtuallyghetto.com/ovf)
[^7]: [A Linux Development Desktop with VMware Horizon - Part I: Horizon](https://rguske.github.io/post/a-linux-development-desktop-with-vmware-horizon-part-i-horizon/)
[^8]: [Download OVFTOOL 4.4.1](https://my.vmware.com/group/vmware/downloads/get-download?downloadGroup=OVFTOOL441)
[^9]: [Package gnome-themes-standard](https://packages.ubuntu.com/search?keywords=gnome-themes-standard)
[^10]: [Package libcanberra-gtk-module](https://packages.ubuntu.com/focal/libcanberra-gtk-module)
[^11]: [A Linux Development Desktop with VMware Horizon - Part III: Shell](https://rguske.github.io/post/a-linux-development-desktop-with-vmware-horizon-part-iii-shell/)

