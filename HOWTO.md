
# Table of Contents
1. [Getting Started](#get_started)
    * [Access](#access)
    * [Login](#login)
    * [File Systems](#file_systems) 
    * [Environment](#environment)
    * [Symbolic Links](#links)
1. [Starting Jobs with Slurm](#slurm)
    * [sbatch](#sbatch)
    * [salloc](#salloc)
    * [GPU Nodes](#gpu_nodes)

There are some tutorials on the TU Berlin Faculty II HPC page, [click here](https://www.tu.berlin/math/hpc-cluster/benutzung). In the following, there are some tips and tricks on how to use the compute cluster specifically in our working group.

## Let's Work on the Cluster <a name="get_started"></a>

### Access <a name="access"></a>
First you need to request access to the cluster. Download the account application form from the [TU Berlin HPC page](https://www.static.tu.berlin/fileadmin/www/40000064/Account-Formulare/Antrag_en.pdf) or this [repository](application_form.pdf). Fill in the required information in the form and send it to your supervisor to sign it (TUB employees are their own supervisor). Then, send (only the first page of) the signed form to clust_staff@math.tu-berlin.de.

### Login <a name="login"></a>
Once your account has been created, you will be notified by the cluster admins via email.
Afterwards, you can access the cluster by *ssh-ing* to the login host:
```
ssh <user_name>@cluster.math.tu-berlin.de
```
Please change the initial password immediately after logging in for the first time (command: `passwd`) !

### Other login hosts

* `cluster-a.math.tu-berlin.de`  *(AMD based)*
* `cluster-i.math.tu-berlin.de`  *(Intel based)*
* `cluster-g.math.tu-berlin.de`  *(GPU development)*
* `cluster-x.math.tu-berlin.de`  *(X2Go)*
* `cluster-v.math.tu-berlin.de`  *(GPU development, visualization)*

### File Systems <a name="file_systems"></a>
        
* `/homes/math/<user_name>` *(directory with quota but daily backup)*
* `/work/<user_name>` *(a lot of storage with quota without backup)*
* `/cfs/projects/milz` *(small dataset directory with 500GB quota without backup)*
* `/net/milz` *(shared folder only accessible for our MILZ working group with 100TB available storage)*
* `/fast/<user_name>` *(fast file system for quick access to temporary data with 50TB quota)*
* `node747:/data` *(huge storage without quota on node747 but only accessible when ssh'ed on node747)*
* `$TMPDIR/` *(temporary directory used in batch jobs)*

Disk Quota (which limits the amount of disk space you can use and the amount of files you can create) can be checked via the command: `Quota`.

**Best practice on what to store where:**
* `/homes/math/<user_name>`: code, settings, tables, other numerical results, ...
* `/work/<user_name>`: environments, packages, cache, input/output, weights, downloads, auxiliary files, ....
* `/cfs/projects/milz`: only **inputs** of **small** datasets that might be shared; datasets which can be processed by the "small" GTX1080 8GB GPUs
* `/net/milz`: data archiving, final outputs, visualizations
* `/fast/<user_name>`: temporary input/output 
* `node747:/data`: all data related to experiments on node747; datasets that only the "big" A100 80GB GPUs can process

**Please note:** It is super important to keep the data at the right places as we share the cluster (thus also some file systems) with others from the math department. Remember as a rule: *If your experiments are only conducted on node747, then please store your data on node747 !* In cases in which jobs of others are killed because of incorrectly storing data, we might simply delete the incorrectly stored data!

### Environment <a name="environment"></a>
There are several modules already installed on the cluster that you can simply load. Most notably [*CUDA*](https://developer.nvidia.com/cuda-toolkit) for GPU accelerated computing. To check what available modules are available:
```
module avail
```
Load a module, e.g. CUDA 12.0:
```
module load cuda/12.0
```
Show loaded modules:
```
module list
```
Unload a module, e.g. CUDA 12.0:
```
module rm cuda/12.0
```

When using **Python** it is recommended to use [Miniconda](https://docs.conda.io/projects/miniconda/en/latest/) to keep your environment on the cluster easy to maintain. Some notes on installing Miniconda can also be found [here in this repository](../python/miniconda.md). Alternatively, you can also use [virtual environments](https://docs.python.org/3/library/venv.html). Please refer to the respective documentation to setup the environment and package manager.

### Symbolic Links <a name="links"></a>
A symbolic link is a special type of file that points to another file or directory on a different file system or partition. This comes in handy when dealing with Quota in your *home* directory.
The general syntax for creating a symbolic link is:
```
ln -s <path_to_the_file/folder_to_be_linked> <the_path_of_the_link_to_be_created>
```
For example, you can move and store all your *Miniconda* files on the *work* file system, but can access from your *home* folder. More specifically, if you want to create a symbolic link from the directory `/work/<user_name>/miniconda3/` to directory `/homes/math/<user_name>/miniconda3` you would run:
```
ln -s /work/<user_name>/miniconda3 /homes/math/<user_name>/miniconda3
```
Any modification to `/homes/math/<user_name>/miniconda3` will also be reflected in the original files in `/work/<user_name>/miniconda3/`. Since the large files are stored on a different file system, this will not affect the quota in your *home* directory.

Some folders for which it is recommended to not store in *home* and create a symbolic link:
* `.cache`
* `.conda`
* `.local`
* `miniconda3`
* `.vscode-server`

Check disk usage in your *home* to identify large files/folders:
```
du -sch .[!.]* * | sort -h
```
There are some links already set up by the cluster admins. Check the folder `/homes/math/<user_name>/.links`. For example, `.conda` could already be pointing to the work disk via `$HOME/.conda -> $HOME/.links/.conda -> /work/<user_name>/.conda`.

In general, you can check if there's a symbolic link and to which directory it is pointing to, using
```
ls -l /path/to/symbolic_link/to_be_checked
```
In case you set a symbolic link wrong and you would like to undo this, simply remove (rm) or unlink the symbolic link, e.g.
```
unlink /path/of/symbolic_link
```

## Slurm <a name="slurm"></a>
[Slurm](https://slurm.schedmd.com/) is the job scheduler running on the cluster. You need to start and schedule your jobs via Slurm. Please refer to the documentation for all configuration options. In the following we cover just a few basics to get going.

===============
### Partitions <a name="ggo_partitions"></a>
Our group Gottschalk partition aka the **Jumbo** has eight [A100](https://www.nvidia.com/de-de/data-center/a100/)s and the corresponding server name is `node747`. Two of the A100 have been partitioned for development and tasks that do not require all of the 80GB memory. The current partitions are:  

* 6x unmodified A100 with 80GB each: -sxm4-80gb (an unmodified A100 has 7 compute instances)
* 2x MIG partition with 3 compute instances and 40GB memory each : _3g.40gb
* 4x MIG partition with 2 compute instances and 20GB memory each : _2g.20gb

The setup can be changed later if needed, but not instantly as it requires modification in the system configuration.

If only a GPU is requested from the batch system, you are likely getting an unpartitioned A100, simply because they're 1st in the list. But if the server is busy, you probably will get one of the MIG devices.
So it makes sense to specify the type explicitly for all jobs:
```
--gres=gpu:nvidia_a100-sxm4-80gb:1 # one unpartitioned A100 (up to 6)
--gres=gpu:nvidia_a100_3g.40gb:1 # one 40GB partition
--gres=gpu:nvidia_a100_2g.20gb:1 # one 20GB partition
```

### sbatch Basics <a name="sbatch"></a>
To run your first jobs on the cluster create your own sbatch-script. Here you can specify your job and the required resources.

[example.sbatch](example.sbatch):
```
#!/bin/bash --login
#SBATCH --job-name=example
#SBATCH --gres=gpu:nvidia_a100-sxm4-80gb:1 
#SBATCH --mem=100g
#SBATCH --time=5:00:00
#SBATCH --partition=ggo

srun python_example.sh
```
In this sbatch example script, called *example*, we request one GPU and 100GB memory for five hours to run the script ```python_example.sh``` on the partition *ggo* (group Gottschalk, the partition with the eight A100s aka the **Jumbo**).

Before scheduling the job, note that the sbatch and sh files need execution permissions:
```
chmod +x example.sbatch python_example.sh
```

Now you can schedule your job:
```
sbatch example.sbatch
```

The script [python_example.sh](python_example.sh) will be executed, which activates the environment and starts [torch_gpu_check.py](torch_gpu_check.py).

Check whether your job is running and find out the job id:
```
[<user_name>@cluster-i ~]$ squeue

             JOBID PARTITION     NAME         USER ST       TIME  NODES NODELIST(REASON)
              1646       ggo interact  <user_name> CF       0:01      1 node747
```
The job will start as soon as the requested resources are available. The status (**ST**) will then change from *configuring* (**CF**, job has been allocated resources, but are waiting for them to become ready for use) to *running* (**R**, job has an allocation):
```
[<user_name>@cluster-i ~]$ squeue

             JOBID PARTITION     NAME         USER ST       TIME  NODES NODELIST(REASON)
              1646       ggo interact  <user_name>  R       0:01      1 node747
```

After the job has started, Slurm will create a file ```slurm-$JOBID.out``` in your working directory to save the output of the sbatch example script.

**Note on `srun` vs. `sbatch`**: 
+ `srun` is interactive and blocking (output written to your terminal, cannot write other commands until it is finished) 
+ `sbatch` is batch processing and non-blocking (output written to a file, other commands can be submitted right away).

To cancel your job:
```
scancel $JOBID
```
**Mail Notifications**:
Be informed on the status of your job via email by adding the following lines to your sbatch script:
```
#SBATCH --mail-user=<user_name>@math.tu-berlin.de
#SBATCH --mail-type=END
```
**Note for Windows users:**
The sbatch file has DOS (Windows) style line endings (\r\n) by default instead of UNIX/Linux style line endings (\n). Linux systems expect UNIX-style line endings, so you need to convert the file to the correct format.

In VSCode, this is easily done by setting the line sequence endings to LF instead of CRLF.

==============
### salloc Basics <a name="salloc"></a>
Slurm manages the access to the nodes and thus also the GPUs. To use a GPU in an interactive session you can "reserve" the required set of resources. 
```
salloc -p ggo --gres=gpu:nvidia_a100-sxm4-80gb:2 --time=1:00:00
```
Here, we start a new shell and allocate two A100s (on `node747`) for one hour in there.

Note that it can take up to *10 minutes* to start the job (as the **Jumbo** consumes ~1200 Watt even while idling. This is why it is turned off after not using the machine for a while and has to wake up again).
If the Slurm job allocation is running, you can access `node747` by *ssh-ing* from the cluster:
```
ssh node747 
```
or directly form your local machine via the cluster as jump proxy:
```
ssh -J <user_name>@cluster.math.tu-berlin.de <user_name>@node747
```

Check if the requested resources have been granted:
```
[<user_name>@node747 ~]$ nvidia-smi

+---------------------------------------------------------------------------------------+
| NVIDIA-SMI 535.104.05             Driver Version: 535.104.05   CUDA Version: 12.2     |
|-----------------------------------------+----------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |         Memory-Usage | GPU-Util  Compute M. |
|                                         |                      |               MIG M. |
|=========================================+======================+======================|
|   0  NVIDIA A100-SXM4-80GB          On  | 00000000:46:00.0 Off |                    0 |
| N/A   26C    P0              62W / 400W |      4MiB / 81920MiB |      0%      Default |
|                                         |                      |             Disabled |
+-----------------------------------------+----------------------+----------------------+
|   1  NVIDIA A100-SXM4-80GB          On  | 00000000:4C:00.0 Off |                    0 |
| N/A   31C    P0              65W / 400W |      4MiB / 81920MiB |      0%      Default |
|                                         |                      |             Disabled |
+-----------------------------------------+----------------------+----------------------+
```
Two **NVIDIA A100-SXM4-80GB** free to use. Super cool!

==============
### GPU Nodes <a name="gpu_nodes"></a>
Besides the big GPUs on `node747`, there are two more GPU nodes defined in Slurm. List of currently available GPUs:
* 8 x A100 GPU with *80* GB memory each on `node747`
* 4 x RTX6000 with *50* GB memory each on `node876`
* 4 x GTX1080 with *8* GB memory each on `node564`
* 1 x H100 GPU with *80* GB memory on `node881`

The GTX1080s on `node564` are well suited for developing code. To use one, e.g. run `salloc`:
```
salloc --gres gpu:gtx1080:1 --<other_options>
```
Similarly, the command for the RTX6000 on `node876`:
```
salloc --gres gpu:rtx_a6000:1 --<other_options>

```
And for the H100 on `node881`:
```
salloc --gres=gpu:h100_pcie_80g:1 --<other_options>
```

If there is a greater demand for more (smaller) GPUs, some more GPUs can be moved to the Slurm batch system. In this case, please contact clust_staff@math.tu-berlin.de and ask politely.

