# SLURM installation steps

* [Prerequisites](#Prerequisites)
  - [Delete failed installation](#delete-failed-installation-of-slurm)
  - [Clock](#clock)
  - [Install MariaDB](#install-MariaDB)
  - [Users](#users)
  - [ssh](#ssh)
* [Install Munge](#install-munge)
  - [Installation](#installation)
  - [Test Munge](#test-munge)
* [Install Slurm](#install-slurm)
* [Configuration](#configuration)
  - [slurm.conf](#slurmconf) 	
  - [Slurm Accounting](#slurm-accounting)
* [Start Services](#start-services)
* [Use Slurm](#use-slurm)

# Prerequisites

## Delete failed installation of Slurm

I leave this optional step in case you tried to install Slurm, and it didn’t work. We want to uninstall the parts related to Slurm unless you’re using the dependencies for something else.

First, I remove the database where I kept Slurm’s accounting.
<pre>yum remove mariadb-server mariadb-devel -y</pre>
Next, I remove Slurm and Munge. Munge is an authentication tool used to identify messaging from the Slurm machines.

<pre>yum remove slurm munge munge-libs munge-devel -y</pre>
I check if the slurm and munge users exist.

<pre>cat /etc/passwd | grep slurm</pre>
Then, I delete the users and corresponding folders.

<pre>userdel - r slurm
userdel -r munge
userdel: user munge is currently used by process 26278
kill 26278
userdel -r munge</pre>
Slurm, Munge, and Mariadb should be adequately wiped. Now, we can start a fresh installation that actually works.

## Clock

Synchronize clock on all nodes of the cluster.

## Install MariaDB

You can install MariaDB to store the accounting that Slurm provides. If you want to store accounting, here’s the time to do so. I only install this on the master node. I use the master node as our SlurmDB node.

<pre>yum install mariadb-server mariadb-devel -y</pre>
You can setup MariaDB later. You just need to install it before building the Slurm RPMs.

## Users

Create a slurm user on *all nodes* of the cluster [1]\
NOTE: The SlurmUser must exist prior to starting Slurm and must exist on all nodes of the cluster.\
Make sure the clocks, users and groups (UIDs and GIDs) are synchronized across the cluster.\
Create the users/groups for slurm and munge, for example, for *all the nodes* before you install Slurm or Munge:
<pre>export MUNGEUSER=991
groupadd -g $MUNGEUSER munge
useradd  -m -c "MUNGE Uid 'N' Gid Emporium" -d /var/lib/munge -u $MUNGEUSER -g munge  -s /sbin/nologin munge
export SLURMUSER=992
groupadd -g $SLURMUSER slurm
useradd  -m -c "SLURM workload manager" -d /var/lib/slurm -u $SLURMUSER -g slurm  -s /bin/bash slurm</pre>
Make sure that these same users are created identically on all nodes. This must be done prior to installing RPMs (which would create random UID/GID pairs if these users don't exist).

We recommend that you create a Unix user slurm for use by slurmctld. This user name will also be specified using the SlurmUser in the slurm.conf configuration file. This user must exist on all nodes of the cluster. Note that files and directories used by slurmctld will need to be readable or writable by the user SlurmUser (the Slurm configuration files must be readable; the log file directory and state save directory must be writable).

## ssh

Set up passwordless ssh between controller and compute nodes.

# Install Munge
## Installation
If you are using CentOS 7, get the latest EPEL repository.

<pre>yum install epel-release</pre>
Install munge on *all nodes*.
<pre>yum install munge munge-libs munge-devel -y</pre>
After installing Munge, you need to create a secret key on the master node. First, we install rng-tools to properly create the key.
<pre>yum install rng-tools -y
rngd -r /dev/urandom</pre>
Now, we create the secret key. You only have to do the creation of the secret key on the master node.
<pre>/usr/sbin/create-munge-key -r</pre>
<pre>dd if=/dev/urandom bs=1 count=1024 > /etc/munge/munge.key
chown munge: /etc/munge/munge.key
chmod 400 /etc/munge/munge.key</pre>
After the secret key is created, you will need to send this key to all of the compute nodes.
<pre>scp /etc/munge/munge.key root@compute-1:/etc/munge
scp /etc/munge/munge.key root@compute-2:/etc/munge
.
.
scp /etc/munge/munge.key root@compute-n:/etc/munge</pre>
Now, we SSH into *every node* and correct the permissions as well as start the Munge service. Chown munge key to user munge. Chmod munge key to 0700.
<pre>chown -R munge: /etc/munge/ /var/log/munge/
chmod 0700 /etc/munge/ /var/log/munge/</pre>
Start the Munge service on all nodes.
<pre>systemctl enable munge
systemctl start munge</pre>
## Test Munge
Then you can test munge. To test Munge, we can try to access another node with Munge from our server node.
<pre>munge -n
munge -n | unmunge
munge -n | ssh compute-2 unmunge
remunge</pre>
If you encounter no errors, then Munge is working as expected.\
Make sure the MUNGE daemon, munged is started before you start the Slurm daemons.\


# Install Slurm
Slurm has a few dependencies that we need to install before proceeding.

<pre>yum install openssl openssl-devel pam-devel numactl numactl-devel hwloc hwloc-devel lua lua-devel readline-devel rrdtool-devel ncurses-devel man2html libibmad libibumad -y</pre>
Now, we download the latest version of Slurm preferably in a shared folder. The latest version of Slurm may be different from our version.
<pre>cd /nfs
wget http://www.schedmd.com/download/latest/slurm-15.08.9.tar.bz2</pre>
To build RPMs directly, copy the distributed tarball into a directory and execute (substituting the appropriate Slurm version number). If you don’t have rpmbuild yet:
<pre>yum install rpm-build</pre>
Then build the RPMs.
<pre>rpmbuild -ta slurm-15.08.9.tar.bz2</pre>
The rpm files will be installed under the $(HOME)/rpmbuild directory of the user building them. For example, for the default user:
<pre>cd /home/centos/rpmbuild/RPMS/x86_64</pre>
Now, we will move the Slurm rpms for installation on the master and computer nodes.
<pre>mkdir /nfs/slurm-rpms
cp slurm-15.08.9-1.el7.centos.x86_64.rpm slurm-devel-15.08.9-1.el7.centos.x86_64.rpm slurm-munge-15.08.9-1.el7.centos.x86_64.rpm slurm-perlapi-15.08.9-1.el7.centos.x86_64.rpm slurm-plugins-15.08.9-1.el7.centos.x86_64.rpm slurm-sjobexit-15.08.9-1.el7.centos.x86_64.rpm slurm-sjstat-15.08.9-1.el7.centos.x86_64.rpm slurm-torque-15.08.9-1.el7.centos.x86_64.rpm /nfs/slurm-rpms</pre>
If you dont have a shared folder scp into the compute nodes:
<pre>cd /home/centos/rpmbuild/RPMS/x86_64</pre>
<pre>scp * root@compute-1:~/
scp * root@compute-2:~/
.
.
scp * root@compute-n:~/</pre>
On every node that you want to be a master and compute node, we install those rpms. 
<pre>yum --nogpgcheck localinstall slurm-15.08.9-1.el7.centos.x86_64.rpm slurm-devel-15.08.9-1.el7.centos.x86_64.rpm slurm-munge-15.08.9-1.el7.centos.x86_64.rpm slurm-perlapi-15.08.9-1.el7.centos.x86_64.rpm slurm-plugins-15.08.9-1.el7.centos.x86_64.rpm slurm-sjobexit-15.08.9-1.el7.centos.x86_64.rpm slurm-sjstat-15.08.9-1.el7.centos.x86_64.rpm slurm-torque-15.08.9-1.el7.centos.x86_64.rpm</pre>
On the master node:
<pre>yum --nogpgcheck localinstall slurm-slurmctld-19.05.4-1.el7.x86_64.rpm slurm-slurmdbd-19.05.4-1.el7.x86_64.rpm</pre>
On the compute nodes:
<pre>yum --nogpgcheck localinstall slurm-slurmd-19.05.4-1.el7.x86_64.rpm</pre>

After we have installed Slurm on every machine, we will configure Slurm properly.
# Configuration
## slurm.conf

Build a configuration file using your favorite web browser and doc/html/configurator.html or alternatively visit http://slurm.schedmd.com/configurator.html to make a configuration file for Slurm.
It is mandatory that slurm.conf is *identical on all nodes*

After you hit Submit on the form, you will be given the full Slurm configuration file to copy.\
On the master node:
<pre>cd /etc/slurm
vim slurm.conf</pre>
Copy the form’s Slurm configuration file that was created from the website and paste it into slurm.conf.\
Executing the command slurmd -C on each compute node will print its physical configuration (sockets, cores, real memory size, etc.), which can be used in constructing the slurm.conf file.\
Save the resulting output to /etc/slurm/slurm.conf.

Now that the master node has the slurm.conf correctly, we need to send this file to the other compute nodes.
<pre>scp slurm.conf root@compute-1:/etc/slurm/slurm.conf
scp slurm.conf root@compute-2:/etc/slurm/slurm.conf
.
.
scp slurm.conf root@compute-n:/etc/slurm/slurm.conf</pre>

Now, according to the settings in the slurm.conf file we need to configure the spool and log directories. These spool directories may need to be be created manually as they are not part of the RPM installation. On the master node:
<pre>mkdir /var/spool/slurmctld
chown slurm: /var/spool/slurmctld
chmod 755 /var/spool/slurmctld
touch /var/log/slurmctld.log
chown slurm: /var/log/slurmctld.log
touch /var/log/slurm_jobacct.log /var/log/slurm_jobcomp.log
chown slurm: /var/log/slurm_jobacct.log /var/log/slurm_jobcomp.log</pre>
On the compute nodes:
<pre>mkdir /var/spool/slurmd
chown slurm: /var/spool/slurmd
chmod 755 /var/spool/slurmd
touch /var/log/slurmd.log
chown slurm: /var/log/slurmd.log</pre>

We must check if the parent directories for Slurm's log files, process ID files, state save directories, etc. are not created by Slurm. If they don't exist they must be created and made writable by SlurmUser as needed prior to starting Slurm daemons.
In slurm.conf it's essential that the important spool directories and the slurm user are defined correctly:
<pre>SlurmUser=slurm
SlurmdSpoolDir=/var/spool/slurmd
StateSaveLocation=/var/spool/slurmctld</pre>

## Slurm Accounting

The MariaDB database will take care of the accounting for slurm. It is necessary to set up the configuration file /etc/slurm/slurmdbd.conf for this to work.

Then create the slurm database:

<pre># mysql -p
mysql> grant all on slurm_acct_db.* TO 'slurm'@'localhost' identified by '[password]' with grant option;
mysql> create database slurm_acct_db;
mysql> quit;</pre>

# Start services

Once the above steps are complete start the required services:

systemctl start slurmd (compute nodes)\
systemctl start slurmdbd (master node)\
systemctl start slurmctld (master node)

# Use slurm

To display the compute nodes:

<pre>scontrol show nodes</pre>

To run jobs on the master node:

<pre>srun /bin/hostname</pre>

To display the job queue:

<pre>scontrol show jobs</pre>


[1]: http://eniac.cyi.ac.cy/display/UserDoc/Copy+of+Slurm+notes
[this]: https://slurm.schedmd.com/configurator.html
[2]: https://slurm.schedmd.com/quickstart_admin.html
