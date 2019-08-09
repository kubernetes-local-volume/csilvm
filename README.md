# CSI plugin for LVM2 [![Issues](https://img.shields.io/badge/Issues-JIRA-ff69b4.svg?style=flat)](https://jira.mesosphere.com/issues/?jql=project%20%3D%20DCOS_OSS%20and%20component%20%3D%20csilvm%20and%20status%20not%20in%20(Resolved%2C%20Closed)) [![Build Status](https://jenkins.mesosphere.com/service/jenkins/buildStatus/icon?job=public-csilvm-pipeline/csilvm-pipelines/master)](https://jenkins.mesosphere.com/service/jenkins/job/public-csilvm-pipeline/job/csilvm-pipelines/job/master/)

This is a [container storage interface (CSI)](https://github.com/container-storage-interface/spec) plugin for LVM2.
It exposes a CSI-compliant API to a LVM2 volume group (VG).
The names of the volume group (VG) and the physical volumes (PVs) it consists of are passed to the plugin at launch time as command-line parameters.
CSI volumes map to LVM2 logical volumes (LVs).

## Getting Started


### Setting up your local environment

You need a properly configured Go installation.
See the `Dockerfile` for the version of Go used in CI.
Newer versions of Go should work.

You can `go get` the source code from GitHub.

```bash
go get -v github.com/mesosphere/csilvm
```

If you want to work on the code I suggest the following workflow.

1. Fork the repository on GitHub.
1. `mkdir -p $GOPATH/src/github.com/mesosphere`
1. `cd $GOPATH/src/github.com/mesosphere`
1. `git clone <your-fork> csilvm`
1. `cd csilvm`
1. `git remote add upstream https://github.com/mesosphere/csilvm.git`
1. `git checkout master`
1. `git branch --set-upstream-to=upstream/master`

You now have the source code cloned locally and the git `origin` set to your fork.

To develop your new feature you can create a new branch and push it to your fork.

1. `git checkout master`
1. `git pull upstream master` (to make sure you're up-to-date)
1. `git checkout -b my-feature-branch`
1. Make changes to `./pkg/lvm`.
1. `git commit -a -m 'lvm: refactored CreateLogicalVolume tests'`
1. `git push origin my-feature-branch`
1. Create a GitHub PR from your branch against `mesosphere/csilvm:master`.


### Building the binary

You need a properly configured Go installation.

The simplest option is to `go get` the project source from GitHub.

```bash
go get -v github.com/mesosphere/csilvm/cmd/csilvm
```


### Running the tests


In order to run the tests you need
* a modern Linux distribution. (Fedora 27)
* sudo rights. This is necessary to create loop devices and (un)mount volumes.
* docker installed. (docker-ce-18.03.1.ce-1.fc27.x86_64.rpm package)
* the `raid1` and `dm_raid` kernel modules must be loaded. If it isn't, run `modprobe raid1 dm_raid` as root.

Then run

```bash
make sudo-test
```

While developing in only one package it is simpler and faster to run only certain tests.

For example, if you're adding new functionality to the `./pkg/lvm` package you can do
:
```bash
cd ./pkg/lvm
go test -c -i . && sudo ./lvm.test -test.v -test.run=TestMyNewFeature
```


## How does this plugin map to the CSI specification?

This plugin is a CSI-compliant wrapper around the normal `lvm2` command-line utilities.
These include `pvcreate`, `vgcreate`, `lvcreate`, etc.

This plugin implements the "headless, unified plugin" design given in the CSI specification.

In particular, the CSI specification lists the architecture as follows:

```
                            CO "Node" Host(s)
+-------------------------------------------+
|                                           |
|  +------------+           +------------+  |
|  |     CO     |   gRPC    | Controller |  |
|  |            +----------->    Node    |  |
|  +------------+           |   Plugin   |  |
|                           +------------+  |
|                                           |
+-------------------------------------------+

Figure 3: Headless Plugin deployment, only the CO Node hosts run
Plugins. A unified Plugin component supplies both the Controller
Service and Node Service.
```

Every instance of this plugin controls a single LVM2 volume group (VG).
Every CSI volume corresponds to a LVM2 logical volume (LV).
The CSI RPCs map to command-line utility invocations on LVM2 physical volumes (PV), a LVM2 volume group (VG) or LVM2 logical volumes (LV).
For the the exact command-line invocations read the source code, starting at https://github.com/mesosphere/csilvm/blob/master/pkg/csilvm/server.go.


## Project structure

This project is split into `./pkg` and `./cmd` directories.

The `./pkg` directory contains logic that may be used from unit tests.

The `./cmd` directory contains commands that interface between the environment (e.g., parsing command-line options, reading environment variables, etc.) and the logic contained in the `./pkg` directory.

The `./pkg` directory is split into the `./pkg/lvm` and `./pkg/csilvm` packages.

The `./pkg/lvm` package provides a Go wrapper around LVM2 command-line utilities and actions.

The `./pkg/csilvm` package includes all the CSI-related logic and translates CSI RPCs to `./pkg/lvm` function calls.

The `./pkg/csilvm/server.go` file should serve as your entrypoint when reading the code as it includes all the CSI RPC endpoints.


## Deployment

The plugin builds to a single executable binary called `csilvm`.
This binary should be copied to the node.
It is expected that the Plugin Supervisor will launch the binary using the appropriate command-line flags.


### Usage

```
$ ./csilvm --help
Usage of ./csilvm:
  -default-fs string
    	The default filesystem to format new volumes with (default "xfs")
  -default-volume-size uint
    	The default volume size in bytes (default 10737418240)
  -devices string
    	A comma-seperated list of devices in the volume group
  -lockfile string
    	The path to the lock file used to prevent concurrent lvm invocation by multiple csilvm instances
  -node-id string
    	The node ID reported via the CSI Node gRPC service
  -probe-module value
    	Probe checks that the kernel module is loaded
  -remove-volume-group
    	If set, the volume group will be removed when ProbeNode is called.
  -request-limit int
    	Limits backlog of pending requests. (default 10)
  -statsd-format string
    	The statsd format to use (one of: classic, datadog) (default "datadog")
  -statsd-max-udp-size int
    	The size to buffer before transmitting a statsd UDP packet (default 1432)
  -statsd-udp-host-env-var string
    	The name of the environment variable containing the host where a statsd service is listening for stats over UDP
  -statsd-udp-port-env-var string
    	The name of the environment variable containing the port where a statsd service is listening for stats over UDP
  -tag value
    	Value to tag the volume group with (can be given multiple times)
  -unix-addr string
    	The path to the listening unix socket file
  -unix-addr-env string
    	An optional environment variable from which to read the unix-addr
  -volume-group string
    	The name of the volume group to manage
```


### Listening socket

The plugin listens on a unix socket.
The unix socket path can be specified using the `-unix-addr=<path>` command-line option.
The unix socket path can also be specified using the `-unix-addr-env=<env-var-name>` option in which case the path will be read from the environment variable of the given name.
It is expected that the CO will connect to the plugin through the unix socket and will subsequently communicate with it in accordance with the CSI specification.


### Locking

Any command-line invocations executed by the `csilvm` process first acquires a
lock on a lockfile. The path to the lockfile can be overridden using the
`-lockfile=` option or the `CSILVM_LOCKFILE_PATH` environment variable. The
locking behaviour can be disabled by setting the `-lockfile=` option to the
empty string. The purpose of locking around command-line invocations is to
prevent multiple `csilvm` processes from executing concurrent `lvm2` commands.
This works around deadlocks in lvm2. For example,
https://jira.mesosphere.com/browse/DCOS_OSS-5434 and
https://github.com/lvmteam/lvm2/issues/23.

By default the lock file is created at `/run/csilvm.lock` so it is assumed that
the `/run` directory exists and is writable by the `csilvm` process.


### Logging

The plugin emits fairly verbose logs to `STDERR`.
This is not currently configurable.


### Metrics

The plugin emits metrics in StatsD format. By default, it uses the
[DogStatsD](http://docs.datadoghq.com/guides/dogstatsd/) format which augments
the standard StatsD format with tags.

The format of the StatsD metrics can be set using the `-statsd-format` flag. It
defaults to `datadog` but can be set to `classic` in order to emit metrics in
standard StatsD format.

Metrics are emitted over UDP. The StatsD server's host and port are read from
environment variables. The names of the environment variables that specify the
StatsD server's host and port can be set using `-statsd-udp-host-env-var` and
`-statsd-udp-port-env-var` flags, respectively.

Metrics are emitted with the prefix `csilvm`.

The following metrics are reported:

- csilvm_uptime: the uptime (in seconds) of the process
- csilvm_requests: number of requests served
	tags:
	  `result_type`: one of `success`, `error`
	  `method`: the RPC name, e.g., `/csi.v0.Controller/CreateVolume`
- csilvm_requests_latency_(stddev,mean,lower,count,sum,upper): the request duration (in milliseconds)
	tags:
	  `method`: the RPC name, e.g., `/csi.v0.Controller/CreateVolume`
- csilvm_volumes: the number of active logical volumes
- csilvm_bytes_total: the total number of bytes in the volume group
- csilvm_bytes_free: the number of bytes available for creating a linear logical volume
- csilvm_bytes_used: the number of bytes allocated to active logical volumes
- csilvm_pvs: the number of physical volumes in the volume group
- csilvm_missing_pvs: the number of pvs given on the command-line but are not found in the volume group
- csilvm_unexpected_pvs: the number of pvs not given on the command-line but are found in the volume group
- csilvm_lookup_pv_errs: the number of errors encountered while looking for pvs specified on the command-line

Furthermore, all metrics are tagged with `volume-group` set to the
`-volume-group` command-line option.

### Runtime dependencies

The following command-line utilties must be present in the `PATH`:

* the various lvm2 cli utilities (`pvscan`, `vgcreate`, etc.)
* `udevadm`
* `blkid`
* `mkfs`
* `file`
* the filesystem listed as `-default-fs` (defaults to: `xfs`)

For RAID1 support the `raid1` and `dm_raid` kernel modules must be available.

This plugin's tests are run in a centos 7.3.1611 container with lvm2-2.02.183 installed from source.
It should work with newer versions of lvm2 that are backwards-compatible in their command-line interface.
It may work with older versions.


### Startup

When the plugin starts it performs checks and initialization.

Note that, as with all software, the source of truth is the code.
The initialization logic lives in the `(*Server).Setup()` function in `./pkg/csilvm/server.go`.

If the `-remove-volume-group` flag is provided the volume group will be removed during `Setup`.
If at that point the volume group is not found, it is assumed that it was successfully removed and `Setup` succeeds.
The PVs are not removed or cleared.

If the `-remove-volume-group` flag is NOT provided the volume group is looked up.
If the volume group already exists, the plugin checks whether the PVs that constitute that VG matches the list of devices provided on the command-line in the `-devices=<dev1,dev2,...>` flag.
Next it checks whether the volume group tags match the `-tag` list provided on the command-line.

If the volume group does not already exist, the plugin looks up the provided list of PVs corresponding to the `-devices=<dev1,dev2,...>` provided on the command-line.
For each, if it isn't already a LVM2 PV, it zeroes the partition table and runs `pvcreate` to initialize it.
Once all the PVs exist, the new volume group is created consisting of those PVs and tagged with the provided `-tag` list.


### Notes


#### Locking

The plugin is completely stateless and performs no locking around operations.
Instead, it relies on LVM2 to lock around operations that are not reentrant.

#### Logical volume naming

The volume group name is specified at startup through the `-volume-group` argument.

Logical volume names are derived from randomly generated, base36-encoded numbers and are prefixed with `csilv`, for example: `csilv9T8s7d3`.

The CO-specified volume name is captured in a LV tag conforming to one of the following formats:

* `VN.<CO-specified-name>`, if the CO-specified name contains *only* characters safe for LVM tags (`A-Z a-z 0-9 + _ . -`).
* `VN+<base64-rawurlencode(CO-specified-name)>`, otherwise. Encoding is performed without padding.

Examples:

* If the CO-specified volume name is `test-volume`, then the generated LV tag is `VN.test-volume`.
* If the CO-specified volume name is `hello volume`, then the generated LV tag is `VN+aGVsbG8gdm9sdW1l`.

#### Logical volume sizes

The `CreateVolume` RPC will attempt to allocate a volume size that both:

* satisfies the requested capacity with the range limits given, and;
* aligns with an LVM extent boundary (LVM default is 4MiB)

The plugin will choose the smallest size within the requested capacity range that aligns to an extent boundary.
If the plugin cannot align on an extent boundary within the requested capacity range, then the `CreateVolume` RPC will return an error.
For example, if the requested capacity is *exactly* 25MiB (RequiredBytes = LimitBytes = 25MiB) then the RPC will fail because 25MiB does not align to the default 4MiB extent boundary.

#### SINGLE_NODE_READER_ONLY

It is not possible to bind mount a device as 'ro' and thereby prevent write access to it.

As such, this plugin does not support the `SINGLE_NODE_READER_ONLY` access mode for a
volume of access type `BLOCK_DEVICE`.

# Issues

This project uses JIRA instead of GitHub issues to track bugs and feature requests.
You may review the [currently open issues](https://jira.mesosphere.com/issues/?jql=project%20%3D%20DCOS_OSS%20and%20component%20%3D%20csilvm%20and%20status%20not%20in%20(Resolved%2C%20Closed)).
You may also create a new [bug](https://jira.mesosphere.com/secure/CreateIssueDetails!init.jspa?pid=14105&issuetype=1&components=20732&customfield_12300=3&summary=CSILVM%3a+bug+summary+goes+here&description=Environment%3a%0d%0dWhat+you+attempted+to+do%3a%0d%0dThe+result+you+expected%3a%0d%0dThe+result+you+saw+instead%3a%0d&priority=3) or a new [task](https://jira.mesosphere.com/secure/CreateIssueDetails!init.jspa?pid=14105&issuetype=3&components=20732&customfield_12300=3&summary=CSILVM%3a+task+summary+goes+here&priority=3).


# Authors

* @gpaul
* @jdef
* @jieyu


# License

This project is licensed under the Apache 2.0 License - see the LICENSE file for details.
