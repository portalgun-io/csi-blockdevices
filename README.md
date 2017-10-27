CSI-BlockDevices [![Build Status](http://travis-ci.org/thecodeteam/csi-blockdevices.svg?branch=master)](https://travis-ci.org/thecodeteam/csi-blockdevices)
-------
CSI-BlockDevices is a Container Storage Interface
([CSI](https://github.com/container-storage-interface/spec)) plugin for locally
attached block devices. Block devices can be exposed to the plugin by
symlinking them into a directory, by default `/dev/disk/csi-blockdevices`. See
sample commands for details.

This project may be compiled as a stand-alone binary using Golang that,
when run, provides a valid CSI endpoint. This project can also be
vendored or built as a Golang plug-in in order to extend the functionality
of other programs.

Installation
-------------

You'll need a working [Go](https://golang.org) installation. From there,
download and installation is as simple as:

`go get github.com/thecodeteam/csi-blockdevices`

This will download the source to `$GOPATH/src/github.com/thecodeteam/csi-blockdevices`,
and will build install the binary `csi-blockdevices` to `$GOPATH/bin/csi-blockdevices`.

Note that this plugin only works on Linux OS.

Starting the plugin
-------------------

In order to execute the binary, you **must** set the env var `CSI_ENDPOINT`. CSI
is intended to only run over UNIX domain sockets, so a simple way to set this
endpoint to a `.sock` file in the same directory as the project is

`export CSI_ENDPOINT=unix://$(go list -f '{{.Dir}}' github.com/thecodeteam/csi-blockdevices)/csi.sock`

With that in place, you can start the plugin
(assuming that $GOPATH/bin is in your $PATH):

```sh
$ ./csi-blockdevices
INFO[0000] .Serve                                        name=csi-blockdevices
```

Use ctrl-C to exit.

You can enable debug logging (all logging goes to stdout) by setting the
`X_CSI_BD_DEBUG` env var. It doesn't matter what value you set it to, just that
it is set. For example:

```sh
$ X_CSI_BD_DEBUG= ./csi-blockdevices
INFO[0000] .Serve                                        name=csi-blockdevices
DEBU[0000] Added Controller Service
DEBU[0000] Added Node Service
^CINFO[0002] Shutting down server
```

For reference, the available env vars are:

| name | purpose | default |
| - | - | - |
| CSI_ENDPOINT | Set path to UNIX domain socket file | n/a |
| X_CSI_BD_DEBUG | enable debug logging to stdout | n/a |
| X_CSI_BD_NODEONLY | Only run the Node Service (no Controller service) | n/a |
| X_CSI_BD_CONTROLLERONLY | Only run the Controller Service (no Node service) | n/a |
| X_CSI_BD_DEVDIR | Directory to search for available block devices | `/dev/disk/csi-blockdevices` |

Note that the Identity service is required to always be running, and that the
default behavior is to also run both the Controller and the Node service

Using the plugin
----------------

All communication with the plugin is done via gRPC. The easiest way to interact
with a CSI plugin via CLI is to use the `csc` tool found in
[GoCSI](https://github.com/thecodeteam/gocsi).

You can install this tool with:

```sh
go get github.com/thecodeteam/gocsi
go install github.com/thecodeteam/gocsi/csc
```

With $GOPATH/bin in your $PATH, you can issue commands using the `csc` command.
You will want to use a separate shell from where you are running the `csi-blockdevices`
binary, and as such you will once again need to do:

`export CSI_ENDPOINT=unix://$(go list -f '{{.Dir}}' github.com/thecodeteam/csi-blockdevices)/csi.sock`

Here are some sample commands:

```sh
$ csc gets
0.0.0
$ csc getp
csi-blockdevices	0.1.0
$ mkdir /dev/disk/csi-blockdevices
$ cd /dev/disk/csi-blockdevices
$ dd if=/dev/zero of=test.img bs=1024 count=102400 #makes 100MiB disk image
$ losetup /dev/loop0 test.img #attaches disk image to /dev/loop0
$ ln -s /dev/loop0 # creates symlink named loop0 -> /dev/loop0
$ csc ls -version 0.0.0
name=loop0
$ touch /mnt/test
$ touch /mnt/test2
$ csc mnt -version 0.0.0 -mode 1 -t xfs -targetPath /mnt/test id=loop0
$ cat /proc/mounts | grep loop
/dev/loop0 /dev/disk/csi-blockdevices/.mounts/loop0 xfs rw,seclabel,relatime,attr2,inode64,noquota 0 0
/dev/loop0 /mnt/test xfs rw,seclabel,relatime,attr2,inode64,noquota 0 0
$ csc mnt -version 0.0.0 -mode 1 -t xfs -targetPath /mnt/test2 id=loop0
$ cat /proc/mounts | grep loop
/dev/loop0 /dev/disk/csi-blockdevices/.mounts/loop0 xfs rw,seclabel,relatime,attr2,inode64,noquota 0 0
/dev/loop0 /mnt/test xfs rw,seclabel,relatime,attr2,inode64,noquota 0 0
/dev/loop0 /mnt/test2 xfs rw,seclabel,relatime,attr2,inode64,noquota 0 0
$ csc umount -version 0.0.0 -targetPath /mnt/test id=loop0
$ cat /proc/mounts | grep loop
/dev/loop0 /dev/disk/csi-blockdevices/.mounts/loop0 xfs rw,seclabel,relatime,attr2,inode64,noquota 0 0
/dev/loop0 /mnt/test2 xfs rw,seclabel,relatime,attr2,inode64,noquota 0 0
$ csc umount -version 0.0.0 -targetPath /mnt/test2 id=loop0
$ cat /proc/mounts | grep loop
$
```
