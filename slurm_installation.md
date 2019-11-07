## Broad overview of SLURM installation
Create a slurm user on all nodes of the cluster \
Synchronize clock on all nodes of the cluster 

<pre>groupadd slurm
useradd -m slurm -g slurm</pre>
Install munge on all nodes _(what is uid, gid?)_\
Generate munge key\
Chown munge key to user munge\
Chmod munge key to 0400\
Copy keys to all machines\
Use [this] to generate the configuration file slurm.conf\
Copy configuration file slurm.conf to all nodes\
Start slurmctl and slurmd on all nodes\

[1]: http://eniac.cyi.ac.cy/display/UserDoc/Copy+of+Slurm+notes
[this]: https://slurm.schedmd.com/configurator.html

