## Overview of SLURM installation
Create a slurm user on all nodes of the cluster [1]\
Synchronize clock on all nodes of the cluster [1]
Make sure the clocks, users and groups (UIDs and GIDs,_(what is uid, gid?)_) are synchronized across the cluster. [2]

<pre>groupadd slurm
useradd -m slurm -g slurm</pre>
Install munge on all nodes. Make sure that all nodes in your cluster have the same munge.key. Make sure the MUNGE daemon, munged is started before you start the Slurm daemons.\
Generate munge key\
Chown munge key to user munge\
Chmod munge key to 0400\
Copy keys to all machines\
Build a configuration file using your favorite web browser and doc/html/configurator.html or alternatively use [this] to generate the configuration file slurm.conf\
Copy configuration file slurm.conf to all nodes\
Start slurmctl and slurmd on all nodes\

[1]: http://eniac.cyi.ac.cy/display/UserDoc/Copy+of+Slurm+notes
[this]: https://slurm.schedmd.com/configurator.html
[2]: https://slurm.schedmd.com/quickstart_admin.html
