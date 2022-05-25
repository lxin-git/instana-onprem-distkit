# Automatic Deploy Instana Distributed Data Store Components

## Sample environment


| Host                  | zookeeper | kafka | clickhouse | cockroachdb | cassandra | elasticsearch | nfs |
|-----------------------|:---------:|:-----:|:----------:|:-----------:|:---------:|:-------------:|:---:|
| bstore-1.fyre.ibm.com | X         | X     | X          |             | X         | X (master)    |     |
| bstore-2.fyre.ibm.com | X         |       | X          | X           | X         | X(data)       |     |
| bstore-3.fyre.ibm.com | X         | X     |            | X           |           | X(data)       | X   |

check for detailed configure in [bstore.ini](inventory/sample/bstore.ini).

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

- `bstore-1` & `bstore-2`

  `lsblk`
  ```console
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

- `bstore-3`:
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


> make sure you auto mount above filesystems via `/etc/fstab`. 


## Quick Start for deployment

1. Review and edit the inventory file in `inventory/sample/inventory.ini` , you can also take `bstore.ini` as an example.

1. Before deployment, we need check the required version of the datastore components, you need download the kubectl plugin for the specific Instana you want to install. Following steps to check all the required version of the datastore components:
   ```shell
   export INSTANA_VERSION="221-3"
   wget https://self-hosted.instana.io/rpm/release/product/rpm/  generic/x86_64/Packages/instana-kubectl-${INSTANA_VERSION}.x86_64.  rpm
   yum install instana-kubectl-${INSTANA_VERSION}.x86_64.rpm
   kubectl-instana -v
   ```
   output shows like below:
   ```console
   [root@bstore-1 ~]# kubectl-instana -v
   kubectl-instana version 221-3   (commit=18856b199394fa509714df476265d7e8c10da335,   date=2022-04-25T14:23:41Z, image=221-3, branch=release)
   
   Required Database Versions:
     * Cassandra:     3.11.10
     * Clickhouse:    21.3.8.76
     * Cockroach:     21.1.7
     * Elasticsearch: 7.16.3
     * Kafka:         2.7.2
     * Zookeeper:     3.6.3
   ```

1. Review and edit the configuration file in  [`inventory/sample/group_vars/all/all.yml`](inventory/sample/group_vars/all/all.yml), all the instana required configuration placed here, check every parameter that marked as `# instana required.`, change them if you want, if not sure how to set the value, let it be the default in the sample.

1. Now, ready to deploy all those components (use `bstore.ini` as inventory file):
   ```
   ansible-playbook -i inventory/sample/bstore.ini start.yml
   ```

Good Luck !


## More Control Options Samples

- Stop all the data stores
  ```
  ansible-playbook -i inventory/sample/bstore.ini control.yml --tags stop
  ```

- Clean all the data stores
  ```
  ansible-playbook -i inventory/sample/bstore.ini control.yml --tags clean
  ```