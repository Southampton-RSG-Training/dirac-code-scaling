---
title: "Running Example Code on a Cluster"
slug: dirac-code-scaling-running-example-code-cluster
teaching: 30
exercises: 20
questions:
- "How can we get our (or others') code running on an HPC resource?"
objectives:
- "Explain what MPI is used for."
- "Load and use software package modules."
- "Compile and run an MPI program."
- "Build and submit a batch submission script for an MPI program."
- "Describe what happens if we specify too few resources in a job script."
keypoints:
- "HPC systems typically use modules to explicitly demarcate dependencies between needed software packages."
- "Use `module avail` to see what software modules you can use on many HPC systems."
- "Use `module load` to make specific software modules we need to run or compile our code accessible for use."
- "Use a login node responsibly by not running anything that requires too many cores, memory, or time to run."
- "Be sure to specify enough (but not too many!) cores your program needs."
---

We've seen in the previous episode how to use a job scheduler on an HPC system to submit basic jobs to a cluster. Now, let's apply what we've learned to some pre-existing "real" code.

Let's assume we're a researcher who has inherited some code from a colleague that we wish to run on an HPC cluster resource. It's been developed to run on multiple processor cores on a machine, and in time, we wish to understand how well this code is able to take advantage of more cores, but first we need to get it running on the cluster. First, we'll obtain the code and get it running locally, before creating a job script to submit it to our Slurm cluster.


## Introduction to a Code Example

Our colleague's code is a trapezoid implementation for calculating π, and can be found at [https://github.com/DiRAC-HPC/HPC-Skills-Pi](https://github.com/DiRAC-HPC/HPC-Skills-Pi). Ordinarily, we'd expect code wish to run on an HPC resource to be more complex to make the most out of such resources, but for the purposes of this lesson, and to ensure our execution times are reasonable, we'll use this instead.

Fortunately our colleague has implemented this code using MPI, which allows it to take advantage of multiple CPU cores on a machine whilst it is running.

> ## How is our program parallelised?
>
> Our π program uses the popular [Message Passing Interface (MPI)](https://en.wikipedia.org/wiki/Message_Passing_Interface) standard to enable communication between each of the parallel portions of code, running on separate CPU cores. It's been around since the mid-1990's and is widely available on many operating systems. MPI also is designed to make use of multiple cores on a variety of platforms, from a multi-core laptop to large-scale HPC resources such as those available on DiRAC. There are many available [tutorials](https://hpc-tutorials.llnl.gov/mpi/) on MPI.
{: .callout}

Let's take a look at the code. Whilst logged into our HPC resource, clone the repo whilst in your home directory, e.g.

~~~
$ cd
$ git clone https://github.com/DiRAC-HPC/HPC-Skills-Pi
~~~
{: .language-bash}

You'll find the following in the `Scaling/MPI` subdirectory:

- `mpi_pi.c`: this is the source code file written in the language C. It contains an implementation of an algorithm for calculating π, and makes use of an **MPI** (or Message Passing Interface) implementation to parallelise the calculation of π across multiple cores. Feel free to take a look at it, but you don't need to understand the implementation itself for the purposes of this lesson!
- `run.sh`: this Bash script first compiles the `mpi_pi.c` code to an executable called `pi`, using `mpicc` (a specialised compiler command for MPI programs). It then runs the compiled Pi code on 1, 2, 4, 8, and finally 16 cores using `mpirun`, which orchestrates the running of our code using MPI over multiple cores.

If we take a look at the script (e.g. by doing `cat run.sh`), we can see it uses `mpicc -o pi mpi_pi.c` to compile it. Once compiled, we should be able to run it. So let's try to compile it manually:

~~~
$ mpicc -o pi mpi_pi.c
~~~
{: .language-bash}

~~~
bash: mpicc: command not found...
~~~
{: .output}

Interestingly, it cannot seem to find the compiler. So how do we get access to one?


## Setting up an Environment to Run our Code

On a typical high-performance computing system, it is seldom the case that the software we want to use is available when we log in. It is installed, but we will need to “load” it before it can run. These packages could be programming language compilers or interpreters, development frameworks or distributions, software packages, or libraries.

A common question is why aren't all these software features accessible immediately? A key reason is that different pieces of software sometimes require different versions of software and libraries. By using a system of *modules*, where each is a self-contained configuration of a software or library package, we can load up a particular software's dependencies explicitly, which avoids any confusion or incompatibility problems. In addition, any dependencies of these modules are also loaded automatically. It also means we can test software against different package versions and implementations separately in a straightforward manner, again, avoiding any confusion as to what is actually being used in any instance. Hence many HPC systems support the loading and unloading of such modules on demand.

> ## So what Modules are Available?
> 
> Helpfully, we can see which modules are available on a particular system by using the command `module avail`. This is particularly useful when developing software for systems we wish to support, since it tells us explicitly what software we can use as well as the versions supported for each.
{: .callout}

We know that our code uses OpenMPI, and that we need to compile it using an MPI compiler. If we do `module avail`, we can see that there are a few implementations of Open MPI available (indicated by `openmpi`). If we use `module avail openmpi`, we can differentiate these more clearly:

~~~
---------------------------------------- /cosma/local/Modules/modulefiles/mpi -----------------------------------------
openmpi/3.0.1(default)  openmpi/4.0.3  openmpi/4.1.1         openmpi/4.1.4     
openmpi/4.0.1           openmpi/4.0.5  openmpi/4.1.1.no-ucx  openmpi/20190429 
~~~
{: .output}

We can see that the most recent version of Open MPI is 4.1.4, so let's use that.

We can `load` this module into our environment using:

~~~
$ module load openmpi/4.1.4
~~~
{: .language-bash}

However, it tells us first that we have a choice to make. It indicates we need a compiler available first, and that the following are available:

~~~
A compiler must be chosen before loading the openmpi module.
Please load one of the following compiler modules:

        gnu_comp/11.1.0
        gnu_comp/9.3.0
        intel_comp/2022.1.2
~~~
{: .output}

Which makes sense, since it turns out `mpicc` needs an underlying compiler to do the actual compilation. Let's load one first, then load the MPI module again:

~~~
$ module load gnu_comp/11.1.0
$ module load openmpi/4.1.4
~~~
{: .language-bash}

Now, we can finally compile and test our program:

~~~
$ mpicc -o pi mpi_pi.c
$ mpirun -np 1 pi
~~~
{: .language-bash}

And we should see something like the following (although the running time may differ):

~~~
np= 1;    Time=4.062785s;    PI=3.14159265459;	Error=0.00000003182
~~~
{: .output}

> ## How Should I use Login Nodes?
> 
> So far we've been testing our codes on a COSMA login node. This is fine for short runs and a bit of code testing, but it's very important to note that we should not run anything too intensive (i.e. using too many cores or memory) or long-running directly on a login node, otherwise this could degrade system performance for other users. So we must always remember to use login nodes responsibly!
{: .callout}

So now we've got our code configured running directly on the resource, let's submit it as a batch job to SLURM.


> ## Create a Batch Submission Script
> 
> Now we understand how to run our Pi code on our HPC resource, we now need to create a SLURM batch script (similarly to how we've done in the last episode) to specify the requirements for our job and run it.
> 
> Create a new job script named `mpi-pi.sh` in the `HPC-Skills-Pi/Scaling/MPI` directory, taking into account it needs to do the following:
>
> - Specify the job requirements using `#SBATCH` parameters, i.e. using 16 cores and let's say a total walltime of no more 1 minute (taking into account it will run over 1, 2, 4, 8, and 16 cores each time).
> - Use `HPC-Skills-Pi/Scaling/MPI` as a working directory.
> - Load the GNU compiler and MPI modules.
> - Run the `run.sh` script.
> 
> > ## Solution
> > 
> > It should look something like the following (with amendments for your own account, partition, and working directory):
> > 
> > ~~~ bash
> > #!/usr/bin/env bash
> > #SBATCH --account yourAccount
> > #SBATCH --partition cosma7
> > #SBATCH --job-name mpi-pi
> > #SBATCH --time 00:01:00
> > #SBATCH --nodes 1
> > #SBATCH --ntasks 16
> > #SBATCH --mem 1M
> > #SBATCH --chdir /cosma/home/yourProject/yourUsername/HPC-Skills-Pi/Scaling/MPI
> > 
> > module load gnu_comp/11.1.0
> > module load openmpi/4.1.4
> > 
> > ./run.sh
> > ~~~
> > {: .language-bash}
>{: .solution}
{: .challenge}


## Submit and Monitor our Job

Now we have our submission script, we can run the job and monitor it until completion:

~~~
$ sbatch mpi-pi.sh
$ squeue -u yourUsername
~~~
{: .language-bash}

Once complete, you should find the output from the job in a slurm output file, for example:

~~~
====
Starting job 5791417 at Thu  8 Sep 14:37:45 BST 2022 for user dc-crou1.
Running on nodes: m7428
====
np= 1;    Time=3.99667s;    PI=3.14159265459;	Error=0.00000003182
np= 2;    Time=2.064242;    PI=3.14159265459;	Error=0.00000003182
np= 4;    Time=1.075068s;    PI=3.14159265459;	Error=0.00000003182
np= 8;    Time=0.687097s;    PI=3.14159265459;	Error=0.00000003182
np=16;    Time=0.349366s;    PI=3.14159265459;	Error=0.00000003182
~~~
{: .output}

So, we can see that as the number of cores increases, the time to run the Pi code diminishes, as we might expect.

> ## What if we Request Too Few CPU cores?
> 
> Try submitting the job with too few CPU cores for the node by changing your submission script accordingly. What happens?
> 
> > ## Solution
> > 
> > Edit the submission script and reduce the `--ntasks` parameter to `8`, for example. Resubmit using `sbatch`, and you should see something like the following in the output file:
> > 
> > ~~~
> > Running on nodes: m7275
> > ====
> > np= 1;    Time=4.050794s;    PI=3.14159265459;	Error=0.00000003182
> > np= 2;    Time=2.036995s;    PI=3.14159265459;	Error=0.00000003182
> > np= 4;    Time=1.055927s;    PI=3.14159265459;	Error=0.00000003182
> > np= 8;    Time=0.550095s;    PI=3.14159265459;	Error=0.00000003183
> > --------------------------------------------------------------------------
> > There are not enough slots available in the system to satisfy the 16 slots that were requested by the application:
> >
> > ./pi
> >
> > Either request fewer slots for your application, or make more slots available for use.
> >
> > A "slot" is the Open MPI term for an allocatable unit where we can launch a process.  The number of slots available are defined by the environment in which Open MPI processes are run:
> >
> > 1. Hostfile, via "slots=N" clauses (N defaults to number of processor cores if not provided)
> > 2. The --host command line parameter, via a ":N" suffix on the hostname (N defaults to 1 if not provided)
> > 3. Resource manager (e.g., SLURM, PBS/Torque, LSF, etc.)
> > 4. If none of a hostfile, the --host command line parameter, or an RM is present, Open MPI defaults to the number of processor cores
> >
> > In all the above cases, if you want Open MPI to default to the number of hardware threads instead of the number of processor cores, use the --use-hwthread-cpus option.
> >
> > Alternatively, you can use the --oversubscribe option to ignore the number of available slots when deciding the number of processes to launch.
> > --------------------------------------------------------------------------
> > ~~~
> > 
> > So here we receive a very descriptive error that we need to allocate more "slots", an Open MPI term which in our case means we need to assign more CPU cores, or `tasks` in SLURM terminology, to our job specification as we would expect. Helpfully, it tells us how many we need to specify!
>{: .solution}
{: .challenge}

{% include links.md %}

