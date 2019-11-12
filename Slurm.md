Content / NOT FINISHED YET

* [Preface](#preface)
* [Prerequisites](#prerequisites)
  - [Munge](#munge)
  - [Slurm RPMs](#slurm-rpms)
  - [Database](#database)
* [Installation](#installation)
  - [Slurm Controller](#slurm-controller)
  - [Slurm Database](#slurm-database)
  - [Slurm Clients](#slurm-clients)
* [Configuration](#configuration)
  - [Logfiles](#logfiles)
    + [Logrotate](#logrotate)
  - [Accounting](#accounting)
  - [Backfilling and Job Priorities](#backfilling-and-job-priorities)
  - [Resource Limits](#resource-limits)
  - [Node Configuration](#node-configuration)
  - [Local Scratch Directory](#local-scratch-directory)
    + [Prolog](#prolog)
    + [Epilog](#epilog)
  - [Network Topology](#network-topology)
  - [Layouts](#layouts)
* [Command HowTo](#command-howto)
  - [scontrol](#scontrol)
  - [squeue](#squeue)
  - [sinfo](#sinfo)
* [Appendix](#appendix)
  - [slurm.conf](#slurm.conf)
  - [slurmdbd.conf](#slurmdbd.conf)

----

# Preface

Slurm is a freely available batch system. Additional support can be bought from [SchedMD](https://slurm.schedmd.com). Please also check the documentation at this site.
Note, this page is meant to deliver a quick means of getting started with slurm, no bells and whistles. It refers to version 17.11.13, currently installed on the CFC cluster.  

Here are some **important** notes on [slurm upgrades](https://slurm.schedmd.com/quickstart_admin.html#upgrade):

* In general, major upgrades should be possible from the past two major releases (e.g. 17.11.x or 18.08.x to 19.05.x).
* "Slurm's MPI libraries may also change if the major release number change, requiring applications be re-linked (behavior may vary depending upon the MPI implementation used and the specific Slurm changes between releases). Locally developed Slurm plugins may also require modification."
* "The libslurm.so version is increased every major release. *So things like MPI libraries with Slurm integration should be recompiled*. Sometimes it works to just symlink the old .so name(s) to the new one, but this has no guarantee of working."

# Prerequisites
## Munge

Slurm prefers munge for authentication. Slurm/Munge users are created upon package installation (Munge: munge munge-devel munge-libs), but userid as well as groupid must be identical on all machines. 
Once munge is installed, create a key either by running "create-munge-key -r" or by using dd (which is way faster). Also note the file permissions:

<pre>
dd if=/dev/urandom bs=1 count=1024 > /etc/munge/munge.key
chown munge: /etc/munge/munge.key
chmod 400 /etc/munge/munge.key
</pre>

The munge key must be available on all nodes. Quick test if everything works:

<pre>
# munge -n | ssh cfc015 unmunge
STATUS:           Success (0)
ENCODE_HOST:      cfc016 (172.20.2.16)
ENCODE_TIME:      2019-09-17 17:42:36 +0200 (1568734956)
DECODE_TIME:      2019-09-17 17:42:36 +0200 (1568734956)
TTL:              300
CIPHER:           aes128 (4)
MAC:              sha1 (3)
ZIP:              none (0)
UID:              root (0)
GID:              root (0)
LENGTH:           0
</pre>

## Slurm RPMs

Just download the slurm sources from schedmd.com and use rpmbuild:

<pre>
# rpmbuild -ta slurm-[version].tar.bz2
</pre>
or from the specfile with some options:
<pre>
# rpmbuild -ba --with lua --with hwloc --with freeipmi slurm-19.05.2.spec 
</pre>

**Note:** There are a lot of dependencies to be installed upfront.

## Database

Install MariaDB... SchedMD advises to tweak mariadb a bit, thus create /etc/my.cnf.d/innodb.cnf and restart mariadb:

<pre>
[mysqld]
innodb_buffer_pool_size=[XYZ]M
innodb_log_file_size=64M
innodb_lock_wait_timeout=900
</pre>

**Note:** "innodb_buffer_pool_size" should be large, i.e. half of the available memory.

Now create the slurm database:
<pre>
# mysql -p
mysql> grant all on slurm_acct_db.* TO 'slurm'@'localhost' identified by '[password]' with grant option;
mysql> create database slurm_acct_db;
mysql> quit;
</pre>

# Installation

Slurm relies on a consistent slurm configuration (slurm.conf) throughout the entire cluster. Probably it's a good idea to provide "slurm.conf" on a shared volume.

### Slurm Controller

The Slurm controller setup can be redundant on two machines. Here, we just deploy the standard version. Installed packages are: 
<pre>
# yum list installed | grep slurm
slurm.x86_64                    17.11.13-2.el7.centos            @slurm         
slurm-devel.x86_64              17.11.13-2.el7.centos            @slurm         
slurm-example-configs.x86_64    17.11.13-2.el7.centos            @slurm         
slurm-libpmi.x86_64             17.11.13-2.el7.centos            @slurm         
slurm-perlapi.x86_64            17.11.13-2.el7.centos            @slurm         
slurm-slurmctld.x86_64          17.11.13-2.el7.centos            @slurm         
slurm-slurmdbd.x86_64           17.11.13-2.el7.centos            @slurm         
slurm-torque.x86_64             17.11.13-2.el7.centos            @slurm 
</pre>

**Note:** "slurm-slurmdbd" is also installed, since it resides on the same node.

### Slurm Database

The slurm database can also be installed on a different system. In this case, just install the corresponding package.

**Note:** In case of a dual headnode setup (with an active and a passive slurm controller), it may be worthwhile to consider running the database on a galera cluster consisting of these nodes.

### Slurm Clients

<pre>
# yum list installed | grep slurm
slurm.x86_64                    17.11.13-2.el7.centos            @slurm         
slurm-example-configs.x86_64    17.11.13-2.el7.centos            @slurm         
slurm-pam_slurm.x86_64          17.11.13-2.el7.centos            @slurm         
slurm-perlapi.x86_64            17.11.13-2.el7.centos            @slurm         
slurm-slurmd.x86_64             17.11.13-2.el7.centos            @slurm         
slurm-torque.x86_64             17.11.13-2.el7.centos            @slurm 
</pre>

# Configuration

## Log Files

Probably the directory "/var/log/slurm" is missing on the controller nodes and needs to be created manually (owner is slurm:slurm).

slurm.conf:
<pre>
SlurmctldLogFile=/var/log/slurm/slurmctld.log
SlurmdLogFile=/var/log/slurmd.log
</pre>

slurmdbd.conf:
<pre>
LogFile=/var/log/slurm/slurmdbd.log
</pre>

### Logrotate

This should work universally, but requires (a) "/var/log/slurm" on the nodes as well.

<pre>
/var/log/slurm/*.log {
        compress
        missingok
        nocopytruncate
        nodelaycompress
        nomail
        notifempty
        noolddir
        rotate 5
        sharedscripts
        size=5M
        create 640 slurm root
        postrotate
                for daemon in $(/usr/bin/scontrol show daemons)
                do
                        killall -SIGUSR2 $daemon
                done
        endscript
}
</pre>

## Accounting

slurm.conf
<pre>
AccountingStorageType=accounting_storage/slurmdbd
</pre>

slurmdbd.conf:
<pre>
AuthType=auth/munge
DbdAddr=[aaa.bbb.ccc.ddd]
DbdHost=[dbdhostname]
SlurmUser=slurm
DebugLevel=4
LogFile=/var/log/slurm/slurmdbd.log
PidFile=/var/run/slurmdbd.pid
StorageType=accounting_storage/mysql
StoragePass=[password]
StorageUser=[password]
StorageLoc=slurm_acct_db
</pre>

In case you expect many jobs (i.e. high-throughput jobs), database queries can get slow. Tweak slurmdbd.conf accordingly, i.e.:

<pre>
PurgeEventAfter=12months
PurgeJobAfter=12months
PurgeResvAfter=2months
PurgeStepAfter=2months
PurgeSuspendAfter=1month
PurgeTXNAfter=12months
PurgeUsageAfter=12months
</pre>

## User Configuration 

Initialize the cluster, i.e.:
<pre>
sacctmgr create cluster cfc
</pre>
Now we're creating associations by setting up some accounts (which may correspond to the linux groups on the system) as well as users or qos:
<pre>
sacctmgr create account qbicgrp fairshare=1000
sacctmgr create account lisaplus fairshare=1000
[sacctmgr create account chem parent=lisaplus fairshare=500]
[sacctmgr create account phys parent=lisaplus fairshare=500]
sacctmgr create account zdv fairshare=10
</pre>

add some users:
<pre>
sacctmgr create user zrslv01 account=qbicgrp fairshare=100
sacctmgr create user zrslv01 account=lisaplus fairshare=100
sacctmgr modify user zrslv01 set defaultaccount=qbicgrp
sacctmgr create coordinator account=qbicgrp user=zrslv01
</pre>

a quality of service:
<pre>
sacctmgr add qos qbicgrp
sacctmgr modify qos qbicgrp set GrpCPUs=50
sacctmgr show qos format=name,priority,GrpCPUs
      Name   Priority
---------- ----------
    normal          0
   qbicgrp          0

</pre>

Check the association tree with "sacctmgr list assoc tree".

## Backfilling and Job Priorities

**Note:** As of version 19.05 the "fair tree" algorithm is the default. 

<pre>
SchedulerType=sched/backfill
PriorityType=priority/multifactor
</pre>

Priorities are computed according to this (full formula see Slurm documentation):

*P*(Job) = *W*(FS) · *P*(FS) + *W*(QOS) · *P*(QOS) + *W*(PART) · *P*(PART) + *W*(WA) · *P*(WA) + *W*(JS) · *P*(JS) + *W*(ASSOC)· *P*(ASSOC)  

**with** FS = Fairshare, QOS = Quality of Service, PART = Partition, WA=Wait Age, JS=Job Size, ASSOC=Association

<pre>
# PriorityUsageResetPeriod=14-0
PriorityCalcPeriod=15
PriorityDecayHalfLife=7-0
PriorityMaxAge=7-0
# PRIORITY FACTORS
PriorityWeightAge=1000
PriorityWeightFairshare=10000
PriorityWeightJobSize=1000
PriorityWeightPartition=1000
PriorityWeightQOS=100
#PriorityWeightTRES
</pre>

## Resource Limits

Resources must be utilized in the most efficient way and resource limits (CPUs, memory) must be enforced (basic limitations for users/groups are set up in “sacctmgr”):

<pre>
TaskPlugin=affinity,cgroup
SelectType=select/cons_res
DefMemPerCPU=500
EnforcePartLimits=ALL
</pre>

“defMemPerCPU” and “DefaultTime=5” (see partition configuration) basically eliminates incomplete job declarations. Constraints are enforced using linux “cgroups” (cgroup.conf):

<pre>
CgroupAutomount=yes
ConstrainCores=yes
ConstrainRAMSpace=yes
</pre>

Intel hyperthreading may be activated on all physical nodes. “Oversubscribe=NO” takes this into account and prevents major performance hits, i.e.:

<pre>
PartitionName=compute Nodes=cfc0[10-16,31] Default=YES OverSubscribe=NO MaxTime=INFINITE DefaultTime=5 MaxTime=3-00 State=UP
</pre>

“MaxTime” also settles the max. walltime limit, which can be overridden by a “qos” configuration.

## Node Configuration

Slurm does a pretty good job auto-detecting node hardware, hence for instance setting the number of CPUs is not mandatory. Note the weight factor - used to influence the order in which nodes are scheduled:

<pre>
# COMPUTE NODES
NodeName=cfc0[10-16] CPUs=20 RealMemory=64000 Sockets=2 CoresPerSocket=10 ThreadsPerCore=2 Weight=1 State=UNKNOWN
NodeName=cfc031 CPUs=20 RealMemory=510000 Sockets=2 CoresPerSocket=10 ThreadsPerCore=2 Weight=100 State=UNKNOWN
</pre>

GPUs are also configured here using the GRES option, i.e. GRES=GPU:2. This however requires an additional configuration file, "gres.conf". Note, newer versions can also be linked against the nvml libraries, allowing to autodetect Nvidia GPU configurations.

There are many other options (i.e. Allow/DenyUser/Groups). Please check the documentation.

## Local Scratch Directory

In contrast to “torque”, slurm does not provide a “on-the-fly” scratch directory, typically “/scratch/<JOBID>”. This can be achieved with prolog and epilog scripts:

slurm.conf:
<pre>
Prolog=/zdv-system/slurm/prolog.d/*
Epilog=/zdv-system/slurm/epilog.d/*
TaskProlog=/zdv-system/slurm/taskprolog.sh
</pre>

### Prolog

<pre>
#!/bin/bash
SLURM_TMP_BASE="/scratch"
if [ -d "$SLURM_TMP_BASE" ] ; then
        /usr/bin/mkdir -p ${SLURM_TMP_BASE}/${SLURM_JOB_ID}
        /bin/chmod 700 ${SLURM_TMP_BASE}/${SLURM_JOB_ID}
        /usr/bin/chown -R $SLURM_JOB_USER:$SLURM_JOB_GROUP ${SLURM_TMP_BASE}/${SLURM_JOB_ID}
else
        echo "print $SLURM_TMP_BASE not available. exiting."
        exit 1
fi
</pre>

### Epilog

<pre>
#!/bin/bash
/usr/bin/rm -rf /scratch/${SLURM_JOB_ID}
</pre>

## Network Topology

simple text file declaring which node connects to which switch, typically IB:
<pre>
# topology.conf
# Switch Configuration
#
# ibswitches
SwitchName=cf-ibswitch01 Nodes=cfc[01-16]
SwitchName=cf-ibswitch02 Nodes=cfc[17-31]
# ibcore
SwitchName=ibcore Switches=ibswitch0[1-2]
</pre>

## Layouts

TBD.

# Command HowTo

There are some cheat sheets out there, i.e. this [one](http://www.physik.uni-leipzig.de/wiki/files/slurm_summary.pdf). The following is a quick intro into some useful commands when running Slurm.

## scontrol

**Note:** The option "--oneliner" is useful for parsing output.

Drain nodes, set them offline/online
<pre>
# scontrol update nodename=cfc[012-016,031] state=drain reason="maintenance"
# scontrol update nodename=cfc[012-016,031] state=idle"
</pre>

Apply configuration changes in slurm.conf (must be cluster-wide them same)
<pre>
# scontrol reconfigure
</pre>

Display the entire Slurm configuration
<pre>
# scontrol show config
</pre>

Show entities, i.e. node, partition, job. For example
<pre>
# scontrol show partitions
PartitionName=compute
   AllowGroups=ALL AllowAccounts=ALL AllowQos=ALL
   AllocNodes=ALL Default=YES QoS=N/A
   DefaultTime=00:05:00 DisableRootJobs=NO ExclusiveUser=NO GraceTime=0 Hidden=NO
   MaxNodes=UNLIMITED MaxTime=UNLIMITED MinNodes=1 LLN=NO MaxCPUsPerNode=UNLIMITED
   Nodes=cfc0[10-16,31]
   PriorityJobFactor=1 PriorityTier=1 RootOnly=NO ReqResv=NO OverSubscribe=NO
   OverTimeLimit=NONE PreemptMode=OFF
   State=UP TotalCPUs=160 TotalNodes=8 SelectTypeParameters=NONE
   DefMemPerNode=UNLIMITED MaxMemPerNode=UNLIMITED
</pre>

## squeue

Batch queue info. 
<pre>
# squeue
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
            132586   compute nf-ganon  qeakr01 CG      20:49      1 cfc016
            130024   compute RSE43_E1  zxmgl18  R 15-02:42:21      1 cfc014
</pre>
Useful options:

| Option | Description |
| ------ | ------ |
| -l | give "long" output |
| -o "[format options]" | device output options, i.e. "%.8i %.9P %.8j %.8u %.8T %.12M %.12l %.4C %.10m %.4D %R" | 
| -u [username] | list only jobs by user |
| -t [type] | types are i.e. RUNNING, PENDING | 
| -T | show admin reservations |
| --start | show estimated start time of pending jobs |

## sinfo

Slurm node and partition info, i.e.
<pre>
# sinfo -l
Thu Sep 19 15:09:49 2019
PARTITION AVAIL  TIMELIMIT   JOB_SIZE ROOT OVERSUBS     GROUPS  NODES       STATE NODELIST
compute*     up   infinite 1-infinite   no       NO        all      2       mixed cfc[011,015]
[...]
</pre>
| Option | Description |
| ------ | ------ |
| -l | give "long" output |
| -N | node info; "-l -N" wil include node features"|


# Appendix
## slurm.conf

<pre>
#
ControlMachine=cfcmgmt
ControlAddr=172.20.2.250
#
#MailProg=/bin/mail
MpiDefault=none
#MpiParams=ports=#-#
ProctrackType=proctrack/cgroup
ReturnToService=1
SlurmctldPidFile=/var/run/slurmctld.pid
#SlurmctldPort=6817
SlurmdPidFile=/var/run/slurmd.pid
#SlurmdPort=6818
SlurmdSpoolDir=/var/spool/slurmd
SlurmUser=slurm
#SlurmdUser=root
StateSaveLocation=/var/spool/slurm
### TaskPlugin=task/none
TaskPlugin=affinity,cgroup
###TaskPluginParam=Cores
GresTypes=gpu
#
# Network Topology (req. topology.conf)
SwitchType=switch/none
###TopologyPlugin=topology/tree
#
# TIMERS
KillWait=30
MinJobAge=300
SlurmctldTimeout=120
SlurmdTimeout=300
#
# Priority
PriorityType=priority/multifactor
PriorityDecayHalfLife=14-0
PriorityMaxAge=14-0
# Priority Factors
PriorityWeightAge=1000
PriorityWeightFairshare=10000
PriorityWeightJobSize=1000
PriorityWeightPartition=1000
PriorityWeightQOS=0

# SCHEDULING
FastSchedule=1
SchedulerType=sched/backfill
SelectType=select/cons_res
SelectTypeParameters=CR_Core_Memory

#
TmpFS=/scratch
#
Prolog=/zdv-system/slurm/prolog.d/*
Epilog=/zdv-system/slurm/epilog.d/*
TaskProlog=/zdv-system/slurm/taskprolog.sh
PrologEpilogTimeout=5
#
# LOGGING AND ACCOUNTING
AccountingStorageType=accounting_storage/slurmdbd
ClusterName=cfc
#JobAcctGatherFrequency=30
JobAcctGatherType=jobacct_gather/linux
#SlurmctldDebug=3
SlurmctldLogFile=/var/log/slurm/slurmctld.log
#SlurmdDebug=3
SlurmdLogFile=/var/log/slurmd.log
#
# COMPUTE NODES
NodeName=cfc0[01-29] CPUs=20 RealMemory=64000 Sockets=2 CoresPerSocket=10 ThreadsPerCore=2 State=UNKNOWN
NodeName=cfc0[30-31] CPUs=20 RealMemory=128000 Sockets=2 CoresPerSocket=10 ThreadsPerCore=2 State=UNKNOWN
PartitionName=compute Nodes=cfc0[01-31] Default=YES MaxTime=INFINITE State=UP
</pre>

## slurmdbd.conf
<pre>
#
# Archive info
#ArchiveJobs=yes
#ArchiveDir="/tmp"
#ArchiveSteps=yes
#ArchiveScript=
#JobPurge=12
#StepPurge=1
#
# Authentication info
AuthType=auth/munge
#AuthInfo=/var/run/munge/munge.socket.2
#
# slurmDBD info
DbdAddr=172.20.2.250
DbdHost=cfcmgmt
#DbdPort=7031
SlurmUser=slurm
#MessageTimeout=300
DebugLevel=4
#DefaultQOS=normal,standby
LogFile=/var/log/slurm/slurmdbd.log
PidFile=/var/run/slurmdbd.pid
#PluginDir=/usr/lib/slurm
#PrivateData=accounts,users,usage,jobs
#TrackWCKey=yes
#
# Database info
StorageType=accounting_storage/mysql
#StorageHost=localhost
#StoragePort=1234
StoragePass=[password]
StorageUser=slurm
StorageLoc=slurm_acct_db
</pre>
