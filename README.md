# Automatic Deploy Instana Distributed Data Store Components

## Sample environment


| Host                  | zookeeper | kafka | clickhouse | cockroachdb | cassandra | elasticsearch | nfs |
|-----------------------|:---------:|:-----:|:----------:|:-----------:|:---------:|:-------------:|:---:|
| tdata-d1.fyre.ibm.com | X         | X     | X          |             | X         | X (master)    |     |
| tdata-d1.fyre.ibm.com | X         |       | X          | X           | X         | X(data)       |     |
| tdata-d1.fyre.ibm.com | X         | X     |            | X           |           | X(data)       | X   |

check for detailed configure in [tdata.ini](inventory/sample/tdata.ini).

## Prerequisite

Setup the disk and file system mount for
```
/mnt/data
/mnt/metrics
/mnt/traces
/mnt/nfs/share
```
You may get system administrator's help on volume/disk allocation and concur.
Here just few samples in my env.

`lsblk`
```
vda             252:0    0  250G  0 disk
├─vda1          252:1    0    1G  0 part /boot
└─vda2          252:2    0  249G  0 part
  ├─centos-root 253:0    0  233G  0 lvm  /
  └─centos-swap 253:1    0   16G  0 lvm  [SWAP]
vdb             252:16   0  500G  0 disk
vdc             252:32   0  500G  0 disk
vdd             252:48   0  500G  0 disk
```
eg.
```
mkdir -p /mnt/data
pvcreate /dev/vdb
vgcreate datavg /dev/vdb
lvcreate -n datalv -l 100%FREE datavg
mkfs.xfs /dev/datavg/datalv
mount /dev/datavg/datalv /mnt/data
```
```
mkdir -p /mnt/metrics
pvcreate /dev/vdc
vgcreate metricsvg /dev/vdc
lvcreate -n metricslv -l 100%FREE metricsvg
mkfs.xfs /dev/metricsvg/metricslv
mount /dev/metricsvg/metricslv /mnt/metrics
```
```
mkdir -p /mnt/traces
pvcreate /dev/vdd
vgcreate tracesvg /dev/vdd
lvcreate -n traceslv -l 100%FREE tracesvg
mkfs.xfs /dev/tracesvg/traceslv
mount /dev/tracesvg/traceslv /mnt/traces
```

tdata-d3:
```
mkdir -p /mnt/data
mkdir -p /mnt/nfs
pvcreate /dev/vdb
vgcreate datavg /dev/vdb

lvcreate -n datalv -L 200G datavg
mkfs.xfs /dev/datavg/datalv
mount /dev/datavg/datalv /mnt/data

lvcreate -n nfslv -L 200G datavg
mkfs.xfs /dev/datavg/nfslv
mount /dev/datavg/nfslv /mnt/nfs
```


## Quick Start for deployment

Review and edit the inventory file in `inventory/sample/inventory.ini` , you can also take `tdata.ini` as an example.

Review and edit the configuration file in `inventory/sample/group_vars/all/all.yml`, all the instana required configuration placed here, check every parameter that marked as `# instana required.`, change them if you want, if not sure how to set the value, let it be the default in the sample.

Now, ready to deploy all those components:
```
ansible-playbook -i inventory/sample/inventory.ini start.yml
```

Good Luck !
