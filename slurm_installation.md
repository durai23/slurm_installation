# Overview of SLURM installation

* [Prerequisites](#Prerequisites)
* [Install Munge](#install-munge)

# Delete failed installation of Slurm

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

# Prerequisites

## Clock

Synchronize clock on all nodes of the cluster [1]

## Users

Create a slurm user on *all nodes* of the cluster [1]\
NOTE: The SlurmUser must exist prior to starting Slurm and must exist on all nodes of the cluster.\
Make sure the clocks, users and groups (UIDs and GIDs,_(what is uid, gid?)_) are synchronized across the cluster. [2]
<pre>groupadd slurm
useradd -m slurm -g slurm</pre>
Another example:\
Create the users/groups for slurm and munge, for example:
<pre>export MUNGEUSER=981
groupadd -g $MUNGEUSER munge
useradd  -m -c "MUNGE Uid 'N' Gid Emporium" -d /var/lib/munge -u $MUNGEUSER -g munge  -s /sbin/nologin munge
export SlurmUSER=982
groupadd -g $SlurmUSER slurm
useradd  -m -c "Slurm workload manager" -d /var/lib/slurm -u $SlurmUSER -g slurm  -s /bin/bash slurm</pre>
and make sure that these same users are created identically on all nodes. This must be done prior to installing RPMs (which would create random UID/GID pairs if these users don't exist).\
Yet another example:\
For all the nodes, before you install Slurm or Munge:
<pre>export MUNGEUSER=991
groupadd -g $MUNGEUSER munge
useradd  -m -c "MUNGE Uid 'N' Gid Emporium" -d /var/lib/munge -u $MUNGEUSER -g munge  -s /sbin/nologin munge
export SLURMUSER=992
groupadd -g $SLURMUSER slurm
useradd  -m -c "SLURM workload manager" -d /var/lib/slurm -u $SLURMUSER -g slurm  -s /bin/bash slurm</pre>

We recommend that you create a Unix user slurm for use by slurmctld. This user name will also be specified using the SlurmUser in the slurm.conf configuration file. This user must exist on all nodes of the cluster. Note that files and directories used by slurmctld will need to be readable or writable by the user SlurmUser (the Slurm configuration files must be readable; the log file directory and state save directory must be writable).

## ssh

Set up passwordless ssh between controller and compute nodes.

# Install Mung

If you are using CentOS 7, get the latest EPEL repository.

<pre>yum install epel-release</pre>

Install munge on *all nodes*.\
<pre>yum install munge munge-libs munge-devel -y</pre>
After installing Munge, you need to create a secret key on the Server.
<write key creation>
After the secret key is created, you will need to send this key to all of the compute nodes
scp /etc/munge/munge.key root@compute-1:/etc/munge
scp /etc/munge/munge.key root@compute-2:/etc/munge
.
.
scp /etc/munge/munge.key root@compute-n:/etc/munge

Now, we SSH into every node and correct the permissions as well as start the Munge service. Chown munge key to user munge. Chmod munge key to 0700.\
<pre>chown -R munge: /etc/munge/ /var/log/munge/
chmod 0700 /etc/munge/ /var/log/munge/</pre>
<pre>systemctl enable munge
systemctl start munge</pre>
Then you can test munge. To test Munge, we can try to access another node with Munge from our server node.
<pre>munge -n
munge -n | unmunge
munge -n | ssh 3.buhpc.com unmunge
remunge</pre>
If you encounter no errors, then Munge is working as expected.\
The MUNGE daemon, munged, must also be started before Slurm daemons.\
Make sure the MUNGE daemon, munged is started before you start the Slurm daemons.\


Install Slurm
Slurm has a few dependencies that we need to install before proceeding.

<pre>yum install openssl openssl-devel pam-devel numactl numactl-devel hwloc hwloc-devel lua lua-devel readline-devel rrdtool-devel ncurses-devel man2html libibmad libibumad -y</pre>
Now, we download the latest version of Slurm preferably in our shared folder. The latest version of Slurm may be different from our version.
<pre>
cd /nfs
wget http://www.schedmd.com/download/latest/slurm-15.08.9.tar.bz2</pre>



To build RPMs directly, copy the distributed tarball into a directory and execute (substituting the appropriate Slurm version number). If you don’t have rpmbuild yet:
<pre>yum install rpm-build
rpmbuild -ta slurm-19.05.1.tar.bz2</pre>
The rpm files will be installed under the $(HOME)/rpmbuild directory of the user building them. For example, for root user:
<pre>cd /root/rpmbuild/RPMS/x86_64</pre>
Now, we will move the Slurm rpms for installation for the server and computer nodes.
<pre>mkdir /nfs/slurm-rpms
cp slurm-15.08.9-1.el7.centos.x86_64.rpm slurm-devel-15.08.9-1.el7.centos.x86_64.rpm slurm-munge-15.08.9-1.el7.centos.x86_64.rpm slurm-perlapi-15.08.9-1.el7.centos.x86_64.rpm slurm-plugins-15.08.9-1.el7.centos.x86_64.rpm slurm-sjobexit-15.08.9-1.el7.centos.x86_64.rpm slurm-sjstat-15.08.9-1.el7.centos.x86_64.rpm slurm-torque-15.08.9-1.el7.centos.x86_64.rpm /nfs/slurm-rpms</pre>
On every node that you want to be a server and compute node, we install those rpms. In our case, I want every node to be a compute node.
<pre>yum --nogpgcheck localinstall slurm-15.08.9-1.el7.centos.x86_64.rpm slurm-devel-15.08.9-1.el7.centos.x86_64.rpm slurm-munge-15.08.9-1.el7.centos.x86_64.rpm slurm-perlapi-15.08.9-1.el7.centos.x86_64.rpm slurm-plugins-15.08.9-1.el7.centos.x86_64.rpm slurm-sjobexit-15.08.9-1.el7.centos.x86_64.rpm slurm-sjstat-15.08.9-1.el7.centos.x86_64.rpm slurm-torque-15.08.9-1.el7.centos.x86_64.rpm</pre>
After we have installed Slurm on every machine, we will configure Slurm properly.\

## slurm.conf

Build a configuration file using your favorite web browser and doc/html/configurator.html or alternatively http://slurm.schedmd.com/configurator.easy.html\
OR\
Visit http://slurm.schedmd.com/configurator.html to make a configuration file for Slurm.
It is mandatory that slurm.conf is identical on all nodes!
For example, you can leave everything default except:

<pre>ControlMachine: server
ControlAddr: 128.197.116.18
NodeName: compute[1-n]
CPUs: 4
StateSaveLocation: /var/spool/slurmctld
SlurmctldLogFile: /var/log/slurmctld.log
SlurmdLogFile: /var/log/slurmd.log
ClusterName: examplecluster</pre>

After you hit Submit on the form, you will be given the full Slurm configuration file to copy.\
On the server node:
<pre>cd /etc/slurm
vim slurm.conf</pre>
Copy the form’s Slurm configuration file that was created from the website and paste it into slurm.conf.\

Install the configuration file in <sysconfdir>/slurm.conf. Or save the resulting output to /etc/slurm/slurm.conf, or the default location in /usr/local/etc/slurm.conf.\

Now that the server node has the slurm.conf correctly, we need to send this file to the other compute nodes.
<pre>scp slurm.conf root@compute-1:/etc/slurm/slurm.conf
scp slurm.conf root@compute-2:/etc/slurm/slurm.conf
.
.
scp slurm.conf root@compute-n:/etc/slurm/slurm.conf</pre>

The parent directories for Slurm's log files, process ID files, state save directories, etc. are not created by Slurm. They must be created and made writable by SlurmUser as needed prior to starting Slurm daemons.\

In slurm.conf it's essential that the important spool directories and the slurm user are defined correctly:\
<pre>SlurmUser=slurm
SlurmdSpoolDir=/var/spool/slurmd
StateSaveLocation=/var/spool/slurmctld</pre>
NOTE: These spool directories must be created manually (see below), as they are not part of the RPM installation.\
NOTE: You will need to install this configuration file on all nodes of the cluster.\
Copy configuration file slurm.conf to all nodes\
Start slurmctl and slurmd on all nodes\

[1]: http://eniac.cyi.ac.cy/display/UserDoc/Copy+of+Slurm+notes
[this]: https://slurm.schedmd.com/configurator.html
[2]: https://slurm.schedmd.com/quickstart_admin.html
