## Broad overview of SLURM installation
Create a slurm user on all nodes of the cluster\
	<pre>groupadd slurm
	useradd -m slurm -g slurm</pre>
Install munge _(what is uid, gid?)_\
Generate munge key\
Chown munge key to user munge\
Chmod munge key to 0400\
Copy keys to all machines\

