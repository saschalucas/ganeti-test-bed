# Introduction
Ganeti test bed is a way to quickly spawn a virtual/test cluster inside a real Ganeti cluster. By using nested virtualization, it is possible to do a wider range of testing and development including things like DRBD or KVM live migration.

The virtual/test cluster consists of one build instance and 3 nodes (master, node02, node03) each with 8GB of RAM. On the build instance you can modify the Ganeti code and quickly test your changes inside the corresponding virtual/test cluster.

# Requirements
In order to use the Ganeti test bed, you need:

* a working Ganeti cluster (working RAPI/`hail`)
* enough cluster resources for at least a single virtual test cluster: 4x 8GB instance RAM and 4x 45G disks
* an instance network using `gnt-network` with `ip=pool` address management
* a OS interface capable of provisioning instances with ready to use networking and SSH key injection (e.g. [instance-guestfish](https://github.com/saschalucas/instance-guestfish))
* a system running ansible: must reach the RAPI of your real cluster and SSH of the virtual nodes (instances from the real cluster's point of view)

# Setup
Test clusters are provisioned by executing an ansible playbook.

First make a copy from `group_vars/all.example` to `group_vars/all`, then edit it to match your setup. To deploy a cluster based on current upstream git / master branch inside a Debian Buster environment simply run:

```bash
ansible-playbook test_cluster.yml -e target_os=debian-bookworm -t setup
```

Please note, that `debian-bookworm` is the variant of your OS interface specified in `group_vars/all`

## single node real cluster
On a single node cluster `hail` may fail. In this case you can specify the node as an extra ansible variable `-e pnode=name`

## other git repos/branches
In order to use other git repos and/or branches specify them as extra ansible variable on the command line:
```bash
# use the stable-3.0 branch from ganeti github organisation
... -e versions=ganeti:stable-3.0

# use a github user's PR feature branch
... -e versions=name:my_feature

# use master branch from ganeti github organisation and additinally someones feature branch for upgrade tests
... -e versions=+name:my_feature

# use a version tag from ganeti github organisation and additinally someones feature branch for upgrade tests
... -e versions=ganeti:v3.0.2+name:my_feature
```

# Destroying
To remove the test cluster change the tag from `setup` to `destroy` (but keep all extra vars on the command line):
```bash
ansible-playbook test_cluster.yml -e target_os=debian-bookworm -t destroy

```

## Speed
The time to spawn a test cluster heavily depends on your environment. If everything goes right, it will take approx. 5 minutes. Tuning knobs are:

* disk template: All instances try to use migration-unsafe caching (`writeback`). Some disk templates silently override cache settings in order to be safe for migration. Try to use non migrateable templates like `plain` or `file`.
* OS interface: if your OSI is slow, try [instance-guestfish](https://github.com/saschalucas/instance-guestfish) (needs ~30s per instance)
* package mirror speed: if your internet connectivity is slow, try to setup a local cache like `apt-cacher-ng`


# Development

## Modify and test source code
To modify the source code ssh into the `build` instance and run:

```bash
cd /tmp/running
git checkout -b your_branch
vi some/code
make
```

To install/test your changes, ssh into the cluster and run:

```bash
gnt-cluster command "cd /tmp/running && make install"
gnt-cluster command "systemctl restart ganeti"
```

You are now ready to test your modifications.

## save your work
If you are ready with your modifications, add your changes to git, commit them and upload it into a remote:

```bash
git fetch --unshallow origin
git add your/changes
git commit --signoff
git remote add my_remote URL
git push --set-upstream my_remote your_branch
```

## Spawn a test VM
```bash
gnt-instance add -t plain -o instance-guestfish+debian-bookworm --disk 0:size=5G --net 0:network=vm-net,ip=pool -B memory=1G,vcpus=1 test.vm
```

## Spawn many fake instances (i.e. for scaling tests)
```bash
gnt-cluster modify --ipolicy-vcpu-ratio 32 --ipolicy-spindle-ratio 128 --ipolicy-bounds-specs min:cpu-count=1,disk-count=1,disk-size=1,memory-size=1,nic-count=1,spindle-use=1/max:cpu-count=8,disk-count=16,disk-size=1048576,memory-size=32768,nic-count=8,spindle-use=12

for i in $(seq -w 1 100); do gnt-instance add -H fake -t drbd --disk 0:size=1M -B memory=1M -o noop --opportunistic-locking --no-wait-for-sync --submit test$i.vm; done
```
