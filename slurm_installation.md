## Overview of SLURM installation
Create a slurm user on all nodes of the cluster [1]\
NOTE: The SlurmUser must exist prior to starting Slurm and must exist on all nodes of the cluster.\
Synchronize clock on all nodes of the cluster [1]
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
Set up passwordless ssh between controller and compute nodes.
Install munge on all nodes. Make sure that all nodes in your cluster have the same munge.key. Make sure the MUNGE daemon, munged is started before you start the Slurm daemons.\
Generate munge key\
Chown munge key to user munge\
Chmod munge key to 0400\
Copy keys to all machines\
The MUNGE daemon, munged, must also be started before Slurm daemons.\
Build a configuration file using your favorite web browser and doc/html/configurator.html or alternatively use [this] to generate the configuration file slurm.conf\
Install the configuration file in <sysconfdir>/slurm.conf. Or save the resulting output to /etc/slurm/slurm.conf, or the default location in /usr/local/etc/slurm.conf\
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
